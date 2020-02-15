---
layout: default
title: RTL-SDR notes
parent: Linux
nav_order: 12
---
# RTL-SDR notes
{: .no_toc }
This is how I used a DVB-T+DVB+FM dongle to scan for various things in my neighborhood.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started



## Cell phone


### IMEI
IMEI (International Mobile Equipment Identity) number of a mobile phone is a 15 digit number unique to a mobile handset. Just key in #*06# on your mobile phone and it will display its IMEI number (on most phones). IMEI numbers of cellular phones connected to a GSM network are stored in a database (EIR - Equipment Identity Register) containing all valid mobile phone equipment.
When a phone is reported stolen or is not type approved, the number is marked invalid.

### ICCID
ICCID (Integrated Circuit Card Identifier) identifies each SIM internationally. It is inscribed on the back of the SIM Card. A full ICCID is 19 or 20 characters. ICCID can be thought of as the serial number of the SIM Card. It is also concidered as Issuers Identification Number. In layman's terms, it is sim cards ID number.

### MEID
MEID (Mobile Equipment Identifier) is also commonly known as an Electric Serial Number (ESN). MEID is a unique 14-digit number that is a unique identifier of a specific cell phone. For easy remembering, a phone's IMEI number minus the last digit if it has both. Any phones that use the Code Division Multiple Access (CDMA) system and have a supported cellular network plan have an MEID number. 

### IMSI on an iphone
IMSI (International Mobile Subscriber Identity) is a unique number, usually fifteen digits, associated with Global System for Mobile Communications (GSM) and Universal Mobile Telecommunications System (UMTS) network mobile phone users. The IMSI is a unique number identifying a GSM subscriber. This number has two parts. The initial part is comprised of six digits in the North American standard and five digits in the European standard. It identifies the GSM operator in a specific country with whom the subscriber holds an account. The second part is allocated by the network operator to uniquely identify the subscriber. It has the format MCC-MNC-MSIN. MCC = Mobile Country Code; MNC = Mobile Network Code; MSIN = sequential serial number.

The IMSI is stored in the Subscriber Identidy Module (SIM) inside the phone and is sent by the phone to the appropriate network. The IMSI is used to aquire the details of the mobile in the Home Location Register (HLR) or the Visitor Location Register (VLR). A mobile user's POTS number (phone number) is associated with their device using the IMSI.


To prevent the subscriber from being identified and tracked by eavesdroppers on a radio interface, the IMSI is rarely transmitted. A randomly generated temporary mobile subscriber identity (TMSI) is sent instead of the IMSI, to ensure that the identity of the mobile subscriber remains confidential and eliminate the need to transfer it in an undeciphered fashion over radio links.

The IMSI is stored as a 64-bit field in the phone's SIM. IMSI is associated with a mobile network interconnecting with other networks, especially CDMA, GSM and EVDO networks. The number is directly provisioned in the phone or in the R-UIM card. 

The three components within IMSI are mobile country code, mobile network code and mobile subscriber identification number. The mobile country code of IMSI has a similar meaning and format to the location area identifier. The mobile network code has the same format and meaning as the location area identifier and is assigned by the government of each country. The mobile subscriber identification number identifies the mobile subscriber and is assigned by the operator.

When mobile stations are powered on, they perform a location update procedure by indicating their IMSI to the network. The first location update procedure is referred to as the IMSI attach procedure. The mobile station also performs location updating to indicate the current location when it moves to a new location area. The location updating message is sent to the new VLR, giving the location information to the subscriber’s HLR. Location updating is periodically performed. The mobile station is not deregistered if after the updating time period the mobile station is not registered. The IMSI detach procedure is performed when a mobile station is powered off to inform the network that it is no longer connected.


On your iPhone, make a call to *#06#, a message should appear with your IMSI number.
888421023563517

## Acknowledgments
* [https://www.quora.com/What-is-the-difference-between-ICCID-IMSI-and-IMEI-numbers](https://www.quora.com/What-is-the-difference-between-ICCID-IMSI-and-IMEI-numbers)
* [https://safeguarde.com/whats-the-difference-between-meid-vs-imei/](https://safeguarde.com/whats-the-difference-between-meid-vs-imei/)
* [https://www.techopedia.com/definition/5067/international-mobile-subscriber-identity-imsi](https://www.techopedia.com/definition/5067/international-mobile-subscriber-identity-imsi)
* [https://www.queryhome.com/tech/5772/what-is-the-difference-between-iccid-and-imsi](https://www.queryhome.com/tech/5772/what-is-the-difference-between-iccid-and-imsi)
* [https://github.com/Oros42/IMSI-catcher](https://github.com/Oros42/IMSI-catcher)
* [https://tools.kali.org/wireless-attacks/kalibrate-rtl](https://tools.kali.org/wireless-attacks/kalibrate-rtl)
* [https://uk.pcmag.com/news-analysis/11593/cdma-vs-gsm-whats-the-difference](https://uk.pcmag.com/news-analysis/11593/cdma-vs-gsm-whats-the-difference)
* [http://www.taylorkillian.com/2013/08/sdr-showdown-hackrf-vs-bladerf-vs-usrp.html](http://www.taylorkillian.com/2013/08/sdr-showdown-hackrf-vs-bladerf-vs-usrp.html)
* [https://sci-hub.tw/10.1145/3307334.3326082](https://sci-hub.tw/10.1145/3307334.3326082)