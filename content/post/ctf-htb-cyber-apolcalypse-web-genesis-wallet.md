---
title: "CTF Writeup: 2022 HTB Cyber Apolcalypse Web Challenge: Genesis Wallet"
date: 2022-05-19T15:27:19-04:00 
description: "CTF Writeup: 2022 HTB Cyber Apolcalypse Web Challenge: Genesis Wallet" # Description used for search engine.
draft: false # Sets whether to render this page. Draft of true will not be rendered.
toc: true # Controls if a table of contents should be generated for first-level links automatically.
tags:
  - ctf
  - nodejs
  - varnish
  - csrf
---

## Summary
Genesis Wallet was one of the harder web challenges in the 2022 Hack the Box (HTB) CTF. Our team composed of Synack Red Team members finished a respectable 21st place, unfortunately we were very close to solving this challenge and literally were about 5 minutes from a successful solve when time expired - so sad!

Personally I thought this was a very clever challenge and probably one of the best designed web challenges in any CTF I've done to date, so I thought I'd share it with interested readers.

## Setup
You are presented with the Genesis Wallet system, an online site that is used to transfer `GTC` tokens from one wallet address to another. You are provided with a set of credentials (username `icarus` and corresponding password) for a wallet and can log in using these credentials - BUT the site is protected by 2FA and requires you to also have an OTP code to log in.

As with the other HTB CTF challenges, we're provided with the full code of the application, which makes finding the path to the flag slightly easier.

## Application Architecture
This application is a NodeJS application running in a Docker container. It is fronted by a Varnish proxy server and has a SQLite database behind it. 

## Finding the Flag
With many of the white box challenges in this CTF, it is fairly easy to locate where the flag is available - the challenge is getting to it in the running web application!

In this case we can look at the `routes/index.js` file and see that the flag is displayed on the main application dashboard under certain conditions:

```js
router.get(/^\/(\w{2})?\/?dashboard/, AuthMiddleware, async (req, res) => {
	let lang = req.params[0];
	if (!lang) lang = 'en';

	return db.getUser(req.user.username)
		.then(user => {
			let flag = null;
			if (user.balance > 1337 && user.username != 'icarus') flag = fs.readFileSync('/flag.txt').toString();
			res.render(`${lang}/dashboard.html`, { user, flag });
```

So we can see that in order to display the flag we need to satisfy the following conditions:

 * Need to have a wallet balance which is substantial (>1337 `GTC`)
* Need to be logged in as a user who is _not_ `icarus` (the credentials we are provided with the challenge)
 
## Initial Setup - New Account
The site supports self registration functionality that proceeds as follows:

 * Register new username / password
 * Log in with username / password
 * Register 2FA QR code with authenticator
 * Confirm OTP from authenticator

When we create a new account, we discover that new users are granted `0.1 GTC` free into their wallet. Based on the market cap of the `GTC` token this is very generous ;)

![Dashboard Image](/images/ctf-2022-htb-genesis/dashboard_new_user.png)

If we need to reach a balance greater than 1337 `GTC` this will require a lot of accounts and a lot of consolidation of "free" `GTC` tokens. There must be a way to get more tokens into an account we control!

## Finding the Tokens
Let's look at how the database is seeded, maybe this will help us find a wallet with a balance? In `database.js`, which is used at startup of the NodeJS applcation to seed the database, we see the following:

```js
	async migrate() {
		let uOTPKey = OTPHelper.genSecret();
		let uAddress = crypto.createHash('md5').update('icarus').digest("hex");
		return this.db.exec(`
			DROP TABLE IF EXISTS users;
			CREATE TABLE users (
				id         INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
				username   VARCHAR(255) NOT NULL UNIQUE,
				password   VARCHAR(255) NOT NULL,
				balance    DOUBLE DEFAULT 0.100,
				otpkey     VARCHAR(255) NULL,
				address    VARCHAR(36) NOT NULL UNIQUE
			);

			INSERT OR IGNORE INTO users (username, password, balance, otpkey, address)
				VALUES ('icarus', 'FlyHighToTheSky', 1337.10, '${uOTPKey}', '${uAddress}');
```

So we see that the user `icarus` has a wallet balance that is seeded with a little more `GTC` tokens than we need to satisfy the first flag criteria. To satisfy the second criteria, we need that balance to be in a wallet which is not controlled by the user `icarus` - so how can we manage a transfer of funds to our wallet?

## Steps to Transfer Funds
After experimenting with the functionality of the site, we learn the flow to send funds from one wallet to another is as follows:

 * Sender intiates a transaction
  * The transaction includes the receiver's wallet address and amount
  * The transaction may include a note, specified in Markdown
 * Sender confirms the transaction with their OTP
 * Receiver can see the transaction on their dashboard and the funds are added to their wallet

The transaction note specified in Markdown seems interesting, and I immediately thought that this could be an easy CSRF via XSS (a theme very common in web CTF challenges), however in this case we face the dreaded DOMPurify as can be seen in the NodeJS function responsible for converting the Markdown to HTML for rendering (located in `helpers/MDHelper.js`):

```js
const showdown        = require('showdown')
const createDOMPurify = require('dompurify');
const { JSDOM }       = require('jsdom');

const conv = new showdown.Converter({
	completeHTMLDocument: false,
  [...snip...]
});
const makeHtml = (md) => {
    return(conv.makeHtml(md));
}
const filterHTML = (content) => {
    html = makeHtml(content);
    window = new JSDOM('').window;
    DOMPurify = createDOMPurify(window);
    return DOMPurify.sanitize(html, {ALLOWED_TAGS: ['strong', 'em', 'img', 'a', 's', 'ul', 'ol', 'li']});
}
```

So we see that the `showdown` converter is used to convert Markdown to HTML, and the result is run through DOMPurify with a very restricted list of supported tags. I hoped that there was some trick here where an older / vulnerable version of DOMPurify was in use (we've seen this in several CTFs recently) but unfortunately this was not the case.

## Framing the Solution: CSRF via `<img>` tag
After thinking about this for a bit, I realized that we had the potential to submit an `<img>` tag with an `src=` attribute that could invoke a `GET`-based CSRF attack, when the receiver (`icarus`) views the transaction.

### Side Quest: Finding `icarus`'s Wallet
Before I could test this theory, I had to solve the practical problem of not knowing what the wallet address for `icarus` was. Fortunately this was easily determined by running the code in the database seeding script in the Node CLI:

```
$ node
> crypto.createHash('md5').update('icarus').digest("hex");
'1ea8b3ac0640e44c27b3cb8a258a87f8'
```

### Next Steps: Finding a CSRF
Fortunately there were not too many routes in this Node application, and I knew I had to find something that was `GET`-based and 2FA-related. After some code review I came upon the code to handle the initial setup (or resetting) of a user's 2FA authenticator application:

```js
router.get(/^\/(\w{2})?\/?(setup|reset)-2fa/, AuthMiddleware, async (req, res) => {
	let lang = req.params[0];
	if (!lang) lang = 'en';
	let otpkey = OTPHelper.genSecret();

	return db.setOTPKey(req.user.username, otpkey)
		.then(() => {
		return res.render(`${lang}/setup-2fa.html`, {otpkey: otpkey, action: req.params[1]});
```

We can see that, when invoked, this URL will generate a new OTP key for the currently logged-in user, save it to the database, and then display the key to them in their browser. Can you spot the critical bug here?

That's right - the new OTP key is saved to the database _before_ being confirmed by the user. This is a pretty bad design because if the user doesn't confirm that they have scanned the QR into their authenticator app, they could be permanently locked out of their account (having reset the OTP without actually confirming that they have it!).

So, in theory this means that we can send a transaction note with the following HTML content to trigger generation of a new OTP key for the user receiving our transaction:

```html
<img src="/reset-2fa">
```

Of course we need to convert this into Markdown for use in the note:

```markdown
![](/reset-2fa)
```

We have a big problem though - how does this help in our quest to steal funds from `icarus`'s wallet?

### Explainer: OTP Keys and 2FA via TOTP
As a very brief explanation of what OTP keys are - most TOTP solutions rely on the creation of a shared secret, which is known by the server and the client. They use the same algorithm, seeded with this secret, to generate (client) and confirm (server) that the time-based codes are correct.

Knowledge of this secret can lead to an attacker being able to generate their own TOTP codes which are guaranteed to be in sync with any other client as well as the server. This is well explained in the [Wikipedia Article on TOTP codes](https://en.wikipedia.org/wiki/Time-based_one-time_password#:~:text=Time%2Dbased%20one%2Dtime%20password%20(TOTP)%20is%20a,(IETF)%20standard%20RFC%206238.).

### Confirming the CSRF
As a quick confirmation, we created a transaction and sent it to the `icarus` wallet address containing an image that pointed to Burp Collaborator. We confirmed that shortly after the transaction was confirmed by the sender, the `<img>` tag was hit. Examining the code of the application, in `bot.js` we can see that there is code to log into the application and view the "transactions" page after every confirmed transaction:

```js
const viewTransactions = async () => {
    try {
		const browser = await puppeteer.launch(browser_options);
		let context = await browser.createIncognitoBrowserContext();
		let page = await context.newPage();

		let token = await JWTHelper.sign({ username: 'icarus', otpkey: true, verified: true });
		await page.setCookie({
			name: "session",
			'value': token,
			domain: "127.0.0.1"
		});
		await page.goto('http://127.0.0.1/transactions', {
			waitUntil: 'networkidle2',
```

So it seems we are onto something with this CSRF idea since there is clearly a headless browser that is intended to view the transaction note.

## The Final Puzzle Piece - Grabbing the OTP secret
This last piece actually took me a while to think through - we need to be able to view the content of the `/reset-2fa` page after it has been requested by `icarus` from our CSRF attack. How would we do this?

For anyone who plays CTFs regularly you know that usually "things are there for a reason" is an axiom you can follow if you get stuck for ideas. It occurred to me to look at the Varnish proxy configuration and then the final piece of the puzzle became clear. Let's look at `config/cache.vcl`, which is the Varnish Cache configuation file, written in VCL, the Varnish language. Particularly, this piece stood out to me:

```
sub vcl_recv {
    # Only allow caching for GET and HEAD requests
    if (req.method != "GET" && req.method != "HEAD") {
        return (pass);
    }
    # get javascript and css from cache
    if (req.url ~ "(\.(js|css|map)$|\.(js|css)\?version|\.(js|css)\?t)") {
        return (hash);
    }
    # get images from cache
    if (req.url ~ "\.(svg|ico|jpg|jpeg|gif|png)$") {
        return (hash);
    }
```

We can see that Varnish is configured to cache any URL matching these regexes for static content. Crucially, it's important to note that the `req.url` value is the _full URL of the request_ and not just the portion before any query string. This means that the following URL will be cached by Varnish:

```
/reset-2fa?ctf.jpg
```

This was the final piece of the puzzle that was needed to grab the OTP secret. Basically, by adding a query string value that will trigger the Varnish caching of the page content, any other request to that same URL should return the _same content_ until the Varnish cache TTL (in this case set to 2 minutes) expires. This should allow us to view the contents of the cached page which was rendered to `icarus` when the CSRF request took place!

This was a brilliant use of reverse cache poisoning, where we are forcing sensitive data to be cached such that an attacker can read it. Truly a brilliant design choice for this challenge!

## Assembling the Final Payload
We now have all the steps requires to complete the mission:

 * Send `icarus` a small transaction with a note containing the Markdown `![](/reset-2fa?123.jpg)`
 * When viewed, this will trigger a CSRF which will store a new OTP key in `icarus`'s database account
 * The content of the request will be (incorrectly) cached by Varnish
 * We can request the same page to see the cached content
 * We can use the OTP secret to configure our TOTP authenticator app to generate codes allowing us to log into the `icarus` account (as we already have the account password)

Seems simple, right? I ran through this all, super excited, sent the request to `http://46.101.25.63:31787/reset-2fa?123.jpg` and.... got my own account 2FA reset - it wasn't the same OTP key that was cached. Why not!??!?!

Turns out I had overlooked a key part of the Varnish configuration at the beginning of the VCL file:

```
sub vcl_hash {
    hash_data(req.url);

    if (req.http.host) {
        hash_data(req.http.host);
    } else {
        hash_data(server.ip);
    }

    return (lookup);
}
```

The `vcl_hash` function is used to determine what elements of a request make it "unique" such that cached results can be looked up and returned (ref. VCL documentat of [hashing](https://varnish-cache.org/docs/trunk/users-guide/vcl-hashing.html)). In this case, we can see that the Varnish hash is computed by:

 * The URL contents
 * The Host header (if present) or server IP (if header is not present)

So in this case, we are not seeing the cached content because the `icarus` bot is accessing it using the localhost URL as we saw above:

```
		await page.goto('http://127.0.0.1/transactions', {
```

Therefore, the request `http://127.0.0.1/reset-2fa` is not equivalent to our request to the external IP address, thus we don't see the cached result. Fortunately this was an easy fix, we simply needed to change the value of the `Host` header in Burp to `127.0.0.1` to ensure we hit the same Varnish hash entry as `icarus` hit:

![Burp Request](/images/ctf-2022-htb-genesis/burp_request_get_cached_content.png)

We can see in the response presumably the same OTP secret that was saved to the database under `icarus`'s account:

```
		<script>
			genQRCode('PFFF2UZEMUGSAN2H');
		</script>
```

## Finishing Up
Now that we have the response and a QR code (we can show this page in the browser to get the QR code), the rest of the exercise was simply to log into `icarus`'s account using the stolen OTP secret, transfer funds to our own wallet, and log into our own wallet to see the flag:

![Flag](/images/ctf-2022-htb-genesis/flag.png)

## Final Thoughts
Overall this was an extremely fun web challenge that required a number of creative solutions to achieve the seemingly obvious initial goal. I appreciate the designer who put this together!