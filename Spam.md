# Click Trajectories: End-to-End Analysis of the Spam Value Chain

See [[Security-Why]].

## Motivation and Overview
Spam-based advertising is a profitable enterprise. Most anti-spam techniques look at how to stop spam from reaching a users inbox, but this obviously doesn't work. How can we look at the economics of spam *holistically*, and once we do, what is the most vulnerable part of this ecosystem? What would be the most effective way to stop spam?

Components:

* **Advertising** : sending actual spam messages to email addresses in order to entice the user to click and potentially buy something. Since most prevention methods have focused on this stage (IP blacklists, shutting down SMTP proxies), spammers must go to lengths circumvent these measures. These include IP prefix hijacking and botnets. It's profitable for botnet operators to rent out their servers to spammers. 
  * BGP IP prefix hijack: steal prefix, send spam messages "from" IPs, then release prefix. This is hard to trace.
  * Botnets: infected machines + command and control server (C+C). Bots may have cached credentials for email accounts from popular providers. This is nice because it's easy to blacklist emails from boutique domains - not GMail though... there's an active underground market for hacked/stolen credentials. The C+C is often hosted on "bullet proof" servers. But different ASes can just blacklist the provider's IP so the C+C can't talk to the bots.
  * Attacker often uses *DNS fast-flux*: register domain name with DNS registrar. Attacker creates list of IPs where the C+C can be located. Binds each IP addr to the DNS-level hostname for short period (300 seconds). *Domain flux* : attacker reigsters multiple DNS names + does DNS fast flux on all of those hostnames.
* **Click support** : The spammer needs some of the recipients to click on a URL, which usually redirects to a site. Many spammers use URLs that redirect to the real site via DNS or HTTP redirection. Spammers often control the redirecting URL but they also use things like URL shorteners. Spammers often buy domains from black market resellers. They can use their own DNS servers or third-parties that promise not to honor takedown requests. They must also use a web server to either proxy or host content. Most spammers operate as advertisers who work as affiliates for online stores, earning a comission of sales. 
  * 350 million spam emails -> 10k clicks -> 28 purchase attempts
* **Realization** : Seller realizes the value by acquiring a payment. This involves a credit card transaction from an *issuing bank* to the *acquiring bank* via a *card association network*. Merchant often uses a *payment processor* which acts as interface from merchant and payment system. Seller then fulfills the order, usually via "drop shipping" which means the seller doesn't have to keep inventory on hand. 
  * Customer -> Merchant -> Payment processor -> Acquiring bank -> [association network] -> Issuing bank that provides payment to acquiring bank
  * Transactions have MCC codes that tell banks the category of the transactions.
  * Visa/Mastercard/Amex charge heavy fines on acquiring bank if the MCC codes are wrong.
  * Keep acquring banks happy - few banks are willing to serve as acquiring banks for spammers

## Example

1. Recipient receives spam message
1. Recipient clicks on URL in message
1. DNS Lookup is performed to the domain registrar (NS) in **Russia**
1. DNS Lookup is performed to the DNS server (A) in **China**
1. HTTP Get performed to web server/proxy in **Brazil**
1. Server receives content from affiliate program in **Russia**
1. User payment goes through user's bank to bank in **Azerbaijan**
1. Order is fulfilled from **India**

## Results

[[img/spam-urls.png]]

After DNS and HTTP redirects, there are only 3M distinct domains from 93M distinct URLs.

[[img/spam-clustering.png]]

By clustering via content, we see that for each cluster there are very few actual affiliate programs being used. Web clusters are formed via content on the web pages themselves.

### Concentration

Spammers need:

* A domain name
* A registrar
* DNS servers
* Web servers
* ISP

Just two registrars server domains for over 20 affiliate programs (out of a hundred or so). Just two/nine ASes host DNS/Web servers for over 20 programs. There's very high concentration.

Over 50% of affiliate programs use just over 8% of or fewer of the registrars and ASes, and 80% use 20% or fewer of the registrars and ASes, which means affiliate programs tend to use the same ASes/registrars for their spam services.

## Conclusion

The set of banks available for such activities is very modest. If American card issuers refused to *settle* transactiosn from these banks, their ability to make money would be significantly hindered.
3 banks serve as acquiring banks for 95% of the spammers. Spammers almost always deliver the promised goods.

Spammers eventually want to sell things to users.

Scammers just want to steal the users' money. Why do so many scam emails involve Nigerian princes (~53%)? Because Nigerian prince emails act as idiot detectors. Stealing from user money is actually quite time intensive for the scammer. 
