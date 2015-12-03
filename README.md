# Email Address

[![Gem Version](https://badge.fury.io/rb/email_address.svg)](http://rubygems.org/gems/email_address)

This gem provides a ruby language library for working with and validating email addresses.

## Installation With Rails or Bundler

If you are using Rails or a project with Bundler, add this line to your application's Gemfile:

    gem 'email_address'

And then execute:

    $ bundle

## Installation Without Bundler

If you are not using Bundler, you need to install the gem yourself.

    $ gem install email_address

Require the gem inside your script.

    require 'rubygems'
    require 'email_address'

## Usage

`EmailAddress` can create a new object to work with the address,
and also has helper to do easy transformations. See the definitions of the
email address forms later in this README for a better understanding of rules
used in this library.

    address = "Clark.Kent+scoops@gmail.com"
    EmailAddress.valid?(address)    #=> true
    EmailAddress.normal(address)    #=> "clark.kent+scoops@gmail.com"
    EmailAddress.canonical(address) #=> "clarkkent@gmail.com"

    email = EmailAddress.new(address) #=> #<EmailAddress::Address:0x007fe6ee150540 ...>
    email.to_s          #=> "clark.kent+scoops@gmail.com"
    email.normalize     #=> "clark.kent+scoops@gmail.com"
    email.canonical     #=> "clarkkent@gmail.com"
    email.original      #=> "Clark.Kent+scoops@gmail.com"
    email.valid?        #=> true
    email.errors        #=> []

#### Advanced Usage

    email.redact        #=> "bea3f3560a757f8142d38d212a931237b218eb5e@gmail.com"
    email.sha1          #=> "bea3f3560a757f8142d38d212a931237b218eb5e"
    email.md5           #=> "c5be3597c391169a5ad2870f9ca51901"
    email.host_name     #=> "gmail.com"
    email.provider      #=> :google
    email.mailbox       #=> "clark.kent"
    email.tag           #=> "scoops"

    email.host.exchanger.first[:ip] #=> "2a00:1450:400b:c02::1a"
    email.host.txt_hash #=> {:v=>"spf1", :redirect=>"\_spf.google.com"}

    EmailAddress.normal("HIRO@こんにちは世界.com")
                        #=> "hiro@xn--28j2a3ar1pp75ovm7c.com"

### Rails (ActiveRecord) Usage

For Rails' ActiveRecord classes, EmailAddress provides an ActiveRecordValidator.

    class User < ActiveRecord::Base
      validates_with EmailAddress::ActiveRecordValidator, field: :email
    end

There is also support for an email address type for Active Record 5.0 and above.

First, you need to register the type in `config/initizers/types.rb`

    require "email_address/email_address_type"
    ActiveRecord::Type.register(:email_address, EmailAddress::Address)
    ActiveRecord::Type.register(:canonical_email_address,
                                EmailAddress::CanonicalEmailAddressType)

Assume the Users table contains the columns "email" and "unique_email". We want to normalize the address in "email" and store the canonical/unique version in "unique_email".

    class User < ActiveRecord::Base
      attribute :email, :email_address
      attribute :unique_email, :canonical_email_address
    end

Here is how the User model works:

    user = User.new(email:"Pat.Smith+registrations@gmail.com",
                    unique_email:"Pat.Smith+registrations@gmail.com")
    user.email        #=> "pat.smith+registrations@gmail.com"
    user.unique_email #=> "patsmith@gmail.com"

#### Validation

The only true validation is to send a message to the email address and
have the user (or process) verify it has been received. Syntax checks
help prevent erroneous input. Even sent messages can be silently
dropped, or bounced back after acceptance. Even conditions such as a
"Mailbox Full" can mean the email address is known but abandoned.

There are different levels of validations you can perform. By default, it will
validate to the "Provider" (if known), or "Conventional" format defined as the
"default" provider. You may pass a a list of parameters to select
which syntax and network validations to perform.

* CHECK_PROVIDER_SYNTAX: Syntax rules by each email provider. If not defined, it
invokes either Conventional or Standard formats as configured.
* CHECK_CONVENTIONAL_SYNTAX: Real-word, Conventional Syntax. This is usually the
format you want.
* CHECK_STANDARD_SYNTAX: RFC-Compliant Syntax. This is usually not what you want,
and can allow garbage email addresses to validate.
* CHECK_DNS: Perform DNS A-Record lookup on domain. Some mis-configured domains
just set up a DNS "A" record and not an "MX" record even though the "A" host
accepts email.
DNS checks will take longer, especially if the domain is not registered.
* CHECK_MX: Perform DNS MX-Record lookup on domain. Require that a MX server
must be configured for the domain.
DNS checks will take longer, especially if the domain is not registered.
* CHECK_CONNECT: Attempt connection to remote mail server. This could be a slow
operation, as it must perform the DNS lookup, then attempt to connect to a remote
server that may be greylisting your connection, especially if it detects too many
bogus connections originating from your IP address.
Most servers will not accept connections from dynamically assigned IP addresses
such as home networks and shared hosting environments like AWS.
Usually, CHECK_MX is what you want instead.
* CHECK_SMTP: Perform SMTP email verification. Performs a DNS lookup, connects to
the mail server, sends an appropriate HELO, MAIL FROM, and RCPT TO command to determine
if the email address is accepted, then disconnects. It can be a very slow process,
cause problems in reputation from your IP address, and may not accurately report
the non-existence of an email address. Additionally, if you have too
many unknown addresses you are verifying, the remote server could
classify it as  "dictionary attack" and block you.
In other words: do not do this unless you
know exactly what you are doing and the ramifications of it.

Example:

    EmailAddress.valid?(address, EmailAddress::CHECK_CONVENTIONAL_SYNTAX,
                        EmailAddress::CHECK_DNS)

#### Comparison

You can compare email addresses:

    e1 = EmailAddress.new("Clark.Kent@Gmail.com")
    e2 = EmailAddress.new("clark.kent+Superman@Gmail.com")
    e3 = EmailAddress.new(e2.redact)
    e1.to_s           #=> "clark.kent@gmail.com"
    e2.to_s           #=> "clark.kent+superman@gmail.com"
    e3.to_s           #=> "bea3f3560a757f8142d38d212a931237b218eb5e@gmail.com"

    e1 == e2          #=> false (Matches by normalized address)
    e1.same_as?(e2)   #=> true  (Matches as canonical address)
    e1.same_as?(e3)   #=> true  (Matches as redacted address)
    e1 < e2           #=> true  (Compares using normalized address)

#### Matching

Matching addresses by simple patterns:

   * Top-Level-Domain:         .org
   * Domain Name:              example.com
   * Registration Name:        hotmail.   (matches any TLD)
   * Domain Glob:              *.exampl?.com
   * Provider Name:            google
   * Mailbox Name or Glob:     user00*@
   * Address or Glob:          postmaster@domain*.com
   * Provider or Registration: msn

Usage:

    e1 = EmailAddress.new("Clark.Kent@Gmail.com")
    e1.matches?("gmail.com") #=> true
    e1.matches?("google")    #=> true
    e1.matches?(".org")      #=> false
    e1.matches?("g*com")     #=> true
    e1.matches?("gmail.")    #=> true
    e1.matches?("*kent*@")   #=> true

## Email Addresses

#### Forms and Terminology

Most of these terms were created during implemtation of this library and are
not (yet) industry standard terms.

**Original**: email address is of the format given by the user.

**Conventional**: Email address format that conforms the the formats supported
by most email service providers. This removes the "Bad Parts" of the email address
specification. This is the format this library supports by default.

  * Lower-case the local and domain part.
  * Tags are kept as they are important for the user.
  * Remove comments and any "bad parts".
  * International Domain Names (IDN) converted to punycode.

**Standard**: Follows the RFC specifications for email addresses.
This keeps the "Bad Parts" as described later.

  * Local parts are kept without editing.
  * Domain names are converted to lower case.
  * International Domain Names (IDN) converted to punycode.

**Normal**: Email Address is converted the the configured format.
This format is what should be used to identify the account.


**Canonical**: Used to uniquely identify the mailbox.
Useful for locating a user who forgets registering with a tag or
with a "Bad part" in the email address.

  * Based on the Conventional form.
  * Address Tags removed.
  * Special characters removed (dots in gmail addresses are not
significant)

**Redacted**: This form is used to store email address fingerprints
instead of the actual addresses. Useful for treating email addresses
as sensitive data and complying with requests to remove the address
from your database and still maintain the state of the account.


  * Format: sha1(canonical_address)@domain
  * Given an email address, the record can be found

 **Reference**: These forms allows you to publicly share an address without
revealing the actual address. While these digests are not guaranteed to be unique,
they are industry standard methods of providing matching addresses without
opening them sup to harvesting.

  * Can be the **MD5** or **SHA1** of the normalized or canonical address
  * Useful for "do not email" lists
  * Useful for cookies that do not reveal the actual account

**Provider**: The name of the service providing the email either an Email Service Provider (Yahoo, Google, MSN), or ISP (Internet Service Provider) such as a
Cable TV or Telephone company.

#### The Good Parts

Email Addresses are split into two parts: the `local` and `host` part,
separated by the `@` symbol, or of the generalized format:

    mailbox+tag@subdomain.domain.tld

The **Mailbox** usually identifies the user, role account, or application.
A **Tag** is any suffix for the mailbox useful for separating and filtering
incoming email. It is usually preceded by a '+' or other character. Tags are
not always available for a given ESP or MTA.

Local Parts should consist of lower-case 7-bit ASCII alphanumeric and these characters:
`-+'.,` It should start with and end with an alphanumeric character and
no more than one special character should appear together.

Host parts contain a lower-case version of any standard domain name.
International Domain Names are allowed, and can be converted to
[Punycode](http://en.wikipedia.org/wiki/Punycode),
an encoding system of Unicode strings into the 7-bit ASCII character set.
Domain names should be configured with MX records in DNS to receive
email, though this is sometimes mis-configured and the A record can be
used as a backup.

This is the subset of the RFC Email Address specification that should be
used.

#### The Bad Parts

Email addresses are defined and redefined in a series of RFC standards.
Conforming to the full standards is not recommended for easily
identifying and supporting email addresses. Among these specification,
we reject are:

* Case-sensitive local parts: `First.Last@example.com`
* Spaces and Special Characters: `"():;<>@[\\]`
* Quoting and Escaping Requirements: `"first \"nickname\" last"@example.com`
* Comment Parts: `(comment)mailbox@example.com`
* IP and IPv6 addresses as hosts: `mailbox@[127.0.0.1]`
* Non-ASCII (7-bit) characters in the local part: `Pelé@example.com`
* Validation by regular expressions like:
```
    (?:[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+/=?^_`{|}~-]+)*
      |  "(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21\x23-\x5b\x5d-\x7f]
          |  \\[\x01-\x09\x0b\x0c\x0e-\x7f])*")
    @ (?:(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9](?:[a-z0-9-]*[a-z0-9])?
      |  \[(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}
           (?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?|[a-z0-9-]*[a-z0-9]:
              (?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21-\x5a\x53-\x7f]
              |  \\[\x01-\x09\x0b\x0c\x0e-\x7f])+)
         \])
```

#### Internationalization

The industry is moving to support Unicode characters in the local part
of the email address. Currently, SMTP supports only 7-bit ASCII, but a
new `SMTPUTF8` standard is available, but not yet widely implemented.
To work properly, global Email systems must be converted to UTF-8
encoded databases and upgraded to the new email standards.

The problem with i18n email addresses is that support outside of the
given locale becomes hard to enter addresses on keyboards for another
locale. Because of this, internationalized local parts are not yet
supported by default. They are more likely to be erroneous.

Proper personal identity can still be provided using
[MIME Encoded-Words](http://en.wikipedia.org/wiki/MIME#Encoded-Word)
in Email headers.


#### Email Addresses as Sensitive Data

Like Social Security and Credit Card Numbers, email addresses are
becoming more important as a personal identifier on the internet.
Increasingly, we should treat email addresses as sensitive data. If your
site/database becomes compromised by hackers, these email addresses can
be stolen and used to spam your users and to try to gain access to their
accounts. You should not be storing passwords in plain text; perhaps you
don't need to store email addresses un-encoded either.

Consider this: upon registration, store the redacted email address for
the user, and of course, the salted, encrypted password.
When the user logs in, compute the redacted email address from
the user-supplied one and look up the record. Store the original address
in the session for the user, which goes away when the user logs out.

Sometimes, users demand you strike their information from the database.
Instead of deleting their account, you can "redact" their email
address, retaining the state of the account to prevent future
access. Given the original email address again, the redacted account can
be identified if necessary.

Because of these use cases, the **redact** method on the email address
instance has been provided.

## Customizing

See `lib/email_address/config.rb` for more options.

#### Adding Rules for a new Email Service Provider

The library ships with the most common set of provider rules. It is not meant
to house a database of all providers, but a separate `email_address-providers`
gem may be created to hold this data for those who need complete rules.

Personal and Corporate email systems are not intended for either solution.
Any of these email systems may be configured locally.

You can change configuration options and add new providers such as:

    EmailAddress::Config.setup do
      provider :github, domains:%w(github.com github.io)
      option   :check_dns, false
    end

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
