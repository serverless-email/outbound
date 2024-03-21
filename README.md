# Using SES for your outbound email
A collection of information/code for sending outbound email via SES

# The Problem for self-hosted email
Setting up and hosting your own email server is not simply technically challenging, but also politically challenging. 

Through no fault of your own, your server's IP address can end up being blacklisted by one of the major providers, and there's 
little to nothing you can do about it. My domain is twenty-five years old and was routinely banned by Hotmail simply because
someone else on my ISP's network was spamming and Hotmail decided it was simpler to just ban traffic from the whole subnet. 

Dealing with theses issues was frustrating and time-consuming. 

The other challenge is if you want to self-host at home, your ISP probably blocks SMTP (TCP port 25) outbound. Even if you
migrate your email server from the cloud to run on cheap hardware at home (my use case), you need to relay your email
someplace else. 

# Let SES be your outbound SMTP server (or relay)
It turns out, AWS SES has a ready-made solution you can use for pennies a month. They manage all the complexity of IP
reputation, protocol compatability, scale, etc. You just have to sign up, prove you're not a spammer (easy!) and 
configure your existing email server to relay messages off it. 

# Instructions (TODO: refactor this to its own topic)

## Step 1: Create an AWS account
An AWS account is basically a billing construct: all the resources you create under it will be paid for by an account. 
Normally this goes without saying, but AWS is so enormous it bears spelling it out explicitly. Since this login is 
able to spend and monitor money, protect it well with a secure password an 2-factor auth. 

Do **not**, under any circumstances, create access keys for this account. You should use it via the console **only**. 

## Step 2: Don't use your AWS account
It's much safer to create a second "admin" account that can access anything except the money. This is the account you should
use day-to-day when you're working with AWS. With your root account, create an account in IAM, give it console access and 
the "AdministratorAccess" permission. This allows the account access to everything except budgeting. 

Again, do **not** create access keys for this account. Console only. 

## Step 3: Create your "Identity"

First create the "Identity" (domain name) for which you'll send email: 
1. [Log in to SES](https://us-east-1.console.aws.amazon.com/ses/home) with your admin account
2. Verify you're in the AWS region that you want in the upper-right of the screen next to your account name
3. Select "Identities" from the navigation bar on the left
4. Click "create identity"
5. For "Identity Type" select "domain"
6. Enter a domain name that you own, check the "use a custom MAIL FROM domain" box, enter a subdomain (it can be
   anything you want; I use `ses`); and uncheck the "Publish DNS records to Route53" (unless you use Route 53
   already there's not much point).
7. You can ignore the "advanced settings" for DKIM
8. Click "Create Identity"

## Step 4: Verify your identity
In order to send mail for a domain, SES needs to know that you actually own that domain. Otherwise, someone could
just sign up for an SES account and start spamming using anyone else's domain, and that would be unfortunate. 

Verification is as simple as inserting some entries in your DNS, which is usually managed for you by the registrar
you used when you created your domain. You can read about how to do that at the 
[helpful article AWS includes](https://docs.aws.amazon.com/console/ses/verified-identities/verify/domain) or you 
can see if AWS can do it for you with [Easy DKIM](https://docs.aws.amazon.com/console/ses/authentication/dkim/easy). 
Some registrars support Easy DKIM, other's require you to set it up manually. 

It's important to set up the DKIM for your domain name, *and* both the SPF and MX records for your custom `MAIL FROM`
domain, and to leave them in place for as long as you use SES. Failure to do so will result in SES ceasing to
accept relays from you, or ceasing to send them with your custom MAIL FROM, which can cause delivery problems. 

It can take some time for SES to scan your domain and find your DKIM records; take a break and go for a walk. 

## Step 5: Exit the sandbox
AWS initially limits your account to 200 sends per day, and while that's probably enough, you should request that they
[remove your account from the sandbox](https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html). It's easy
and takes your account limit to 50,000 per day and 14 per second, which is plenty enough for me. 

## Step 6: Get SMTP credentials
For your mail server to relay messages of SES, it needs to authenticate itself. To do this,
1. On the navigation bar on the left, select "SMTP Settings"
2. Click the button "Create SMTP Credentials". This will create a new IAM user that can only send email.
3. Give the user a name, and click "Create User"
4. Copy the credentials into your password manager; you won't see them again (but you can create new ones)

## Step 6: Configure your SMTP mail server to use SES
This step will vary depending on the mail server you currently use. I use postfix as a mail server, so for me this meant
adding a line to `main.cf`:
```
relayhost = [email-smtp.us-east-1.amazonaws.com]:587
```
And creating a file with the AWS credentials:
```
[email-smtp.us-east-1.amazonaws.com]:587 op://k8s/SES SMTP credentials/username:op://k8s/SES SMTP credentials/password
```

(That's the actual file I keep on my desktop machine and in Git. When I deploy it, I use a script to inject the actual credentials from 
my password manager.)

## Step 7: Test
Now send a message to another account using your mail server. On the other machine, you can look at the "Received" headers and you should
see some from SES at or near the top:
```
Received: from CH3PR20MB7492.namprd20.prod.outlook.com (2603:10b6:610:1e5::6)
 by SA1PR20MB4442.namprd20.prod.outlook.com with HTTPS; Thu, 21 Mar 2024
 15:15:00 +0000
Received: from BLAP220CA0030.NAMP220.PROD.OUTLOOK.COM (2603:10b6:208:32c::35)
 by CH3PR20MB7492.namprd20.prod.outlook.com (2603:10b6:610:1e5::6) with
 Microsoft SMTP Server (version=TLS1_2,
 cipher=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384) id 15.20.7386.31; Thu, 21 Mar
 2024 15:14:56 +0000
Received: from BL6PEPF0001AB56.namprd02.prod.outlook.com
 (2603:10b6:208:32c:cafe::35) by BLAP220CA0030.outlook.office365.com
 (2603:10b6:208:32c::35) with Microsoft SMTP Server (version=TLS1_2,
 cipher=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384) id 15.20.7386.27 via Frontend
 Transport; Thu, 21 Mar 2024 15:14:56 +0000
Authentication-Results: spf=pass (sender IP is 54.240.8.38)
 smtp.mailfrom=ses.koe.hn; dkim=pass (signature was verified)
 header.d=koe.hn;dkim=pass (signature was verified)
 header.d=amazonses.com;dkim=fail (signature did not verify)
 header.d=koe.hn;dmarc=pass action=none header.from=koe.hn;compauth=pass
 reason=100
Received-SPF: Pass (protection.outlook.com: domain of ses.koe.hn designates
 54.240.8.38 as permitted sender) receiver=protection.outlook.com;
 client-ip=54.240.8.38; helo=a8-38.smtp-out.amazonses.com; pr=C
Received: from a8-38.smtp-out.amazonses.com (54.240.8.38) by
 BL6PEPF0001AB56.mail.protection.outlook.com (10.167.241.8) with Microsoft
 SMTP Server (version=TLS1_2, cipher=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384) id
 15.20.7409.10 via Frontend Transport; Thu, 21 Mar 2024 15:14:55 +0000
DKIM-Signature: v=1; a=rsa-sha256; q=dns/txt; c=relaxed/simple;
	s=ansznc5v3thz4wrg6a3bifpjg5dpvalc; d=koe.hn; t=1711034094;
	h=Date:From:To:Message-ID:Subject:MIME-Version:Content-Type;
	bh=CcSoSASBMYw2iJMxjz3kcl6caQzYORpSg1SRN7XC/Kw=;
	b=HCdPDbtJCPeUYL3HeeNrs7r6mgJOEGEA1y4rsNFgM0B2lgFqCcB1o8J9VA8w4NHk
	MEhwQMnBIqSYQezvYTY9ihQ7vVL58gIhpmmPa3kcUsDCUYloRjwNzKIBxweHncn91XW
	REvWPkjh+HcYu/O2E/C5mS5xyXeWAFNyRbCNIYsPR/PNMww4O0uH2yItxcZ5bqW0eZh
	qQAQTAkUsm41+/0qZ3ewys2s+z4R1pydv92Ts/gXlU8X+8kqldtPcWWshaglq1HKHTT
	m+c6WTvZ+cmQF8cvQpV6fWKL/eRFJMygeIt6eZe9uMyfVmWKZN2Wkv43TWmf4ODkg5R
	Hrj2gLSjIQ==
DKIM-Signature: v=1; a=rsa-sha256; q=dns/txt; c=relaxed/simple;
	s=ug7nbtf4gccmlpwj322ax3p6ow6yfsug; d=amazonses.com; t=1711034094;
	h=Date:From:To:Message-ID:Subject:MIME-Version:Content-Type:Feedback-ID;
	bh=CcSoSASBMYw2iJMxjz3kcl6caQzYORpSg1SRN7XC/Kw=;
	b=UtPqGTiK+SzgzCwScqistCtNtHr8sun2x2XEqhIeMvtuFU6vgUaj5ClF4hrZkgyD
	T6hMqoZvoLLZLryzDyZPSV3iD3koW3qwxEF+5wO9f8tnohRb76OPgi5Z2NzV9SLBKJ3
	c6kGX53Cb6foCF4RB+/NpCAN0oQdTSR3tdvQN62w=
```
1. The top three `Received` lines are Microsoft's internal mail handling
2. The `Authentication-Results` line show Microsoft successfully verifying that
   1. My SPF passed for my `MAIL FROM` domain;
   2. My AWS DKIM passed (it broke the DKIM from my mail server; that's ok)
3. The `Received-SPF` again shows that SPF is working
4. The fourth `Received` header shows the email going from Amazon SES
   being delivered to Microsoft

If you see similar results (a `Received` header from SES and valid DKIM), then
you're all set! You can use this indefinitely for (currently) $0.10 per thousand
emails per month, and I've never had an email rejected by a recipient. 
