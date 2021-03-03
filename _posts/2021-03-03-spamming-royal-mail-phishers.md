---
layout: single
title: Spamming Royal Mail phishers with disinformation
---
Recently, there has been a scourge of phishing domains purporting to be the UK's Royal Mail in an attempt to scam Britons out of their personal details.
You can see [here](https://phishstats.info:8443/public/dashboard/096f9d39-441c-44d2-abbd-ea065e31c74a?search=royal%20mail%20group&start=2020-05-01) (courtesy of PhishStats) that the malicious actors started this attack in February, and have typically registered and hosted these websites with Namecheap.

Let's spam them, shall we?

# Annoying the phishers

The idea here is to spam with disinformation; tons of fake data, to drown out legitimate data and deter identity theft. Before we write a spam script, we must first observe the data being sent. Note: sometimes malicious actors compromise legitimate websites and abuse them to host their phishing pages, so please take care you are not unwittingly carrying out a DoS on some poor small business.

## Kowalski, analysis!

First, it asks for a phone number:
{% include figure image_path="/assets/images/royalmaildisinfo/user.php.png" alt="A web page purporting to be by Royal Mail prompting for the victim's phone number." caption="Figure 1: the first page; `/user.php`." %}
Following that, it then asks for personal details:
{% include figure image_path="/assets/images/royalmaildisinfo/verifylogin.php.png" alt="Another web page, this time prompting for the victim's personal details." caption="Figure 2: the second page; `/verifylogin.php`." %}
Last but certainly not least, it then asks for payment details:
{% include figure image_path="/assets/images/royalmaildisinfo/secure.php.png" alt="Another web page, prompting for the victim's payment details." caption="Figure 3: the third page; the 'aptly-named' `/secure.php`." %}
Then after that, it is the usual "You are now verified!" and the ol' trusty "redirect-to-the-real-page-and-pretend-nothing-happened".

Something of note about these forms: up until the very last form, each form submits to the next page, and the POST data seemingly does nothing except generate hidden form inputs which would continue the cycle. Apparently, only `/secure.php` (the form source excerpt of which is shown below), is the only "important" form, as it POSTs to `/complete.php`:
```html
<form class="user-login-form" data-drupal-selector="user-login-form" data-msg-required="This field is required." action="complete.php?&sessionid=$hash&securessl=true" method="POST" id="user-login-form" accept-charset="UTF-8" autocomplete="off">
  <input type="hidden" name="phone" value="071111111111">
  <input type="hidden" name="fname" value="Chewsday">
  <input type="hidden" name="day" value="06">
  <input type="hidden" name="month" value="Month">
  <input type="hidden" name="year" value="2000">
  <input type="hidden" name="address" value="88">
  <input type="hidden" name="city" value="Bruh">
  <input type="hidden" name="postcode" value="1989">
  <input type="hidden" name="county" value="Bedfordshire">
  <input type="hidden" name="mmn" value="Mememe">
```
(Not even a cookie or session variable, smh)

Following through the "steps" to verify while having Developer Tools (with the network logs, no Burp required!), we can observe the POST payload to `/complete.php` (below) and subsequently spam the endpoint (kindly ignore my testing data, pls and thank):
```
phone=071111111111&fname=Chewsday&day=06&month=Month&year=2000&address=88&city=Bruh&postcode=1989&county=Bedfordshire&mmn=Mememe&ccname=Bruh&ccnum=4111+1111+1111+1111&expiry=03%2F25&cvv=000&acct=69696969&sort=11-11-11&op=Submit
```

Let's get fakin'.

## Much ado about fake identities...

I really don't want to write a random name, date-of-birth, address, credit card and bank details generator, and neither do you. Luckily, Pythonistas have a library for doing all that (and a whole lot more!): [`from faker import Faker`](https://github.com/joke2k/faker). This particular library even allows you to specify locales, such that it would generate UK addresses, UK sort codes and account numbers, etc.

Unfortunately, it isn't quite as simple as plugging it into `requests.post`; there are formatting issues abound, and it would be ideal if the fake data is indistinguishable from authentic input. Here are some examples of my workarounds:

### Phone numbers
As implied in Figure 1, UK phone numbers always start with 07 and are 11 digits long. When you ask Faker for a phone number, it spits out different formats, none of which being the one we want:
```python
>>> from faker import Faker
>>> fake = Faker("en_GB")
>>> [fake.phone_number() for _ in range(5)]
['(0306) 999 0881', '(028) 9018 0419', '+44(0)141 4960350', '+44(0)113 496 0124', '0118 496 0092']
```
The janky alternative is to use the [MSISDN](https://en.wikipedia.org/wiki/MSISDN) like so:
```python
>>> "07" + fake.msisdn()[4:]
'07124037640'
```
### Street address
With Faker, street addresses are two lines, whereas the form only expects one:
```python
>>> fake = Faker("en_GB")
>>> fake.street_address()
'Flat 01V\nAnne alley'
>>> fake.street_address()
'Flat 89\nGilbert lakes'
>>> fake.street_address()
'070 Marie freeway'
>>> fake.street_address()
'Studio 00\nLeah squares'
```
A simple `.replace("\n", "")` would fix this.

## Putting it all together
After some time debugging and testing, this is what I came up with.
{% gist 03208dcba446b30abfa88ffcb11a8b18 %}

# Eviscerating the phishers
All well and good, but what if you found another phishing attempt, and actually want to stop, not hinder, these operations?

There are many routes to take, and it would do well to pursue all of them:

1. [Report it to Google for Chrome and Firefox](https://safebrowsing.google.com/safebrowsing/report_phish/?hl=en). If all goes well, after some time, your browser would pop up with an anti-phishing warning upon navigating to it, preventing victims from falling for the scam.
2. Report it to the domain registrar. Doing a quick `$ whois domain-name.here | grep "Registrar Abuse Contact Email"` should quickly provide you with a point of contact to report the website for abusive activity (namely, phishing).
3. Report it to the server host. Search for the domain's "Autonomous System" (this can usually be done with [urlscan.io](https://urlscan.io)) and use your 1337 OSINT skillz to find how you can file an abuse complaint.
