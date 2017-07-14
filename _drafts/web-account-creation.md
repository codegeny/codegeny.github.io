---
layout: post
title: Web account creation
author: xavier
tag:
  - web
---

As an exercise, I wanted to see how handling website account via cryptographic functions would work.

I would like to share my register/login/remember-me/recover-lost-password/unlock/logout processes with you and hope to get feedback on what I may have done wrong or where they fall short.

## Some notes

* Everything happens in HTTPS
* Cookies are all secured (https-only)
* I never want to disclose if an account exists or not for a given email address, so some error messages may indicate multiple possible failure reasons without precising which one
* I wanted to avoid storing things in DB as much as possible so I tried to use cryptographic signatures and caching when possible

## Registration process

* The registration form has an email field and a captcha
  * The captcha is there to avoid spamming an address with activation links (registration occurs only once so it is a one-time annoyance for legit users)
* If the captcha was not solved correctly, return to form with an error message indicating the captcha solution was not valid
* If the email address is not valid, return to form indicating the email address is not valid
* If the captcha and email are valid, a page will be shown to the user saying that a 24h activation link was sent to the given email address if no account was already registered for that address
* If an account is already registered for that address, stop here
* Send an email with an activation link containing (email, timestamp, signature = base64(signed(hashed('register', email, timestamp))))
  * I'm using SHA1 to hash the ('register', email, timestamp) tuple
  * I'm using Elliptic Curve Digital Signature Algorithm with a 240bits key to sign the hash
  * Nothing is stored in database at this point
* When the user clicks the activation link, verify the following points in that order:
  * If the timestamp is older than 24h, report that the activation request has expired
  * If the signature is not valid (`!verify(hashed('register', email, timestamp), unbase64(signature))`), report that the activation code is not valid
  * If the account already exists, report that this email address is already registered
* If the activation link was successfully verified, present the user with a password form (+ other mandatory information to create an account like full name, birthdate...)
  * The form should contain the (email, timestamp, signature) parameters from the activation link
    * That information can be stored in hidden fields or the form can `POST` to an URL containing that information
* When submitting the form:
  * Verify that password satisfies your criteria (size, complexity), if you have 2 password fields (confirmation), verify that they match
    * Warn the user about any error regarding his password, redisplay the form
  * Verify any other information needed to create the account
    * Warn the user about any error, redisplay the form
  * Verify once again that (email, timestamp, signature) is valid like when the user clicked on the activation link
    * Warn the user that (this registration request has expired) or (the account already exists) or (the code activation code is not valid), don't show the form anymore
* If all verifications succeeded, create the user account in DB
  * Hash the password with BCrypt
  * Automatically log the user in

## Login process

* When accessing the login page, check if the site is undergoing a brute-force attack (someone is trying common passwords on all accounts)
  * The site is under attack if the number of global unsuccessful login attempts have reached a threshold you have defined in a given period of time (say 1000 logins/second and 99% are unsuccessful)
    * I store that information in a round-robin database (see RRDTool) so that size remains constant
  * If it does, add a captcha to the following form and inform the user about what is going on
* The login form has 2 fields: email address and password (+ a captcha in case of brute-force attack)
* When submitting the form, verify if:
  * An account with the given email address exists
  * The account is not locked due to many rapid-fire login attempts (someone is trying all passwords on the current account)
    * I store that information in a cache (not in DB) whose elements have a time-to-live of 30 minutes, an element is defined as {key => account_id/email, value => { last_attempt_timestamp, attempt_count }}
    * An account is locked if there exists an element in cache for that account_id and (current_timestamp - last_attempt_timestamp) in seconds is lower than (2 ^ attempt_count)
      * You can have your own strategy, mine is to wait 1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 1800 (30 mins = TTL) seconds on try 1, 2, 3, 4, 5, 6...
  * The account password and submitted password must match
    * If they don't match:
      * Look up in the cache described above:
        * If there is already a record for that account, increment the attempt_count, update last_attempt_timestamp to current_timestamp
        * If there's no record, create one with { last_attempt_timestamp = current_timestamp, attempt_count = 0 }
      * Increment the number of global unsuccessul login attemps in your round-robin database
  * If any of the above points fails, warn the user with all the following points (without precising which one failed):
    * The password might be incorrect, provide a link to the lost-password page
    * The email address might not be registered, provide a link to the registration page
    * The account is locked (multiple unsuccessfull login attempts), provide a link to the unlock page
    * Redisplay the form
* If all verifications succeeded, log the user in

## Remember me

* Optionally, the login form may contain a remember-me checkbox
* On successful login, a cookie containing the (account_id/email, timestamp, signature = base64(signed(hashed('remember', account_id/email, bcrypted-password, timestamp)))) should be created
* When an unauthenticated user accesses a page with a remember-me cookie, verify the following:
  * the timestamp must not be older than (let's say) one month
  * the account must exist
  * the signature can be verified = verify(hashed('remember', account-id/email, bcrypted-password, timestamp), unbase64(signature))
* If all verifications succeeded, log the user in

## Lost password/recovery process

* The recovery form has an email field and a captcha
  * The captcha is there to avoid spamming an address with recovery links (recovering an account is only needed when someone lost his password)
* If the captcha was not solved correctly, the user is warned about that, return to form
* If the email address is not valid, the user is warned about that, return to form
* If the captcha and email are correct, a page will be shown to the user saying that a 1h recovery link was sent to the given email address if an account exists for that address
* If an account does not exist for that address, stop here
* Send an email with a recovery link containing (account_id/email, timestamp, signature = base64(signed(hashed('recover', account_id/email, bcrypted-password, timestamp))))
* When the user clicks the unlock link, verify the following:
  * the timestamp must not be older than 1h
  * an account exists for that email address/account_id
  * the signature can be verified = verify(hashed('recover', account-id/email, bcrypted-password, timestamp), unbase64(signature))
  * If any of the points above fails, warn the user that (this unlock request has expired) or (the account does not exist) or (the password has already been changed) or (the recovery code is not valid) without precising which one and stop here
* If the recovery link was successfully verified, present the user with a password form
  * The form should contain the (account_id/email, timestamp, signature) from the recovery link
    * That information can be stored in hidden fields or the form can POST to an URL containing that information
* When submitting the form:
  * Verify that password satisfies your criteria (size, complexity), if you have 2 password fields (confirmation), verify that they match
    * Warn the user about any error regarding his password, redisplay the form
  * Verify once again that (account_id/email, timestamp,  signature) is valid like when the user clicked on the recovery link
    * Warn the user that (this registration request has expired) or (the account already exists) or (the code activation code is not valid) without precising which one and stop here (don't show the form anymore)
* If all verifications succeeded:
  * Hash the password with BCrypt and save it in database
  * Automatically log the user in

## Unlock process

This process allows a user to log-in through a link received via email.

* The unlock form has an email field and a captcha
  * The captcha is there to avoid spamming an address with unlock links (unlocking an account is only needed when someone is trying to crack your account or when the site is undergoing brute force attack)
* If the captcha was not solved correctly, the user is warned about that, return to form
* If the email address is not valid, the user is warned about that, return to form
* If the captcha and email are correct, a page will be shown to the user saying that a 1h unlock link was sent to the given email address if an account exists for that address
* If an account does not exist for that address, stop here
* Send an email with an unlock link containing (account_id/email, timestamp, signature = base64(signed(hashed('unlock', account_id/email, bcrypted-password, timestamp))))
* When the user clicks the unlock link, verify the following:
  * the timestamp must not be older than 1h
  * an account exists for that email address/account_id
  * the signature can be verified = verify(hashed('unlock', account-id/email, bcrypted-password, timestamp), unbase64(signature))
  * If any of the points above fails, warn the user that (this unlock request has expired) or (the account does not exist) or (the unlock code is not valid) without precising which one and stop here
* If all verifications succeeded, log the user in

## Logout process

* The logout form has only one button and an anti-CSRF token hidden field
  * The token is defined as base64(signed(hashed('logout', session_id, timestamp)))
    