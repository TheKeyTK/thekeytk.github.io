---
title: Phishing with fake meeting invite
layout: post
---

Have you ever thought about how teams or google meet invites work?

Recently I was tasked with a social engineering engagement and a random thought crosses my mind, how meeting invites actually work and could I abuse it somehow. 

Of course, it turns out someone already thought of that and this method had been used in the wild, but no one explains how it actually works.
Since I couldn't find any blog that technically describes this attack I dig into it myself and in this blog post I will try to explain to you how it works and how you as a pentester/red teamer can include it into your offensive techniques.

### What was my goal for this phishing attack?

Send a fake meeting invite and create a sense of urgency so the target doesn't pay much attention and click my link which asks for credentials.

The usual use case for creating a teams meeting (in outlook) works like this:
- click the "New teams meeting" button
- new email page opens with an email body pre-populated with teams template and meeting URL
- set meeting title and select attendees
- click send and magically your attendees receive a fancy looking email where they can accept or deny a meeting invite


*Creating a meeting in outlook*:
![create_meeting](/assets/create_meeting.png)

*Meeting invite arrived in outlook*:
![meeting_invite](/assets/meeting_invite.png)

### So what actually happened and why does this invite look so fancy?
When you create a meeting invite you are simply sending a normal email with the iCalendar object as an attachment.
Depending on the receiver's client and how it processes the iCalendar object, you get a fancy-looking email or just a simple email with an attachment.
In the image below you can see how it looks like when you receive teams meeting invite in protonmail.

![protonmail_receives_outlook_meeting_invite](/assets/protonmail_receives_outlook_meeting_invite.png)


### What exactly is an iCalendar object?
According to  [wikipedia](https://en.wikipedia.org/wiki/ICalendar){:target="_blank"}:

"The Internet Calendaring and Scheduling Core Object Specification (iCalendar) is a media type which allows users to store and exchange calendaring and scheduling information such as events, to-dos, journal entries, and free/busy information. Files formatted according to the specification usually have an extension of .ics"

iCalendar is supported by Outlook, Google calendar, Yahoo calendar, Apple calendar, and many more. 
In this post, I am focusing on outlook.


Below you can see the iCalendar object of the teams meeting invite. I have simply sent a meeting invite to my protonmail address and just downloaded it and read the attachment.

```
BEGIN:VCALENDAR
METHOD:REQUEST
PRODID:Microsoft Exchange Server 2010
VERSION:2.0
BEGIN:VTIMEZONE
TZID:GTB Standard Time
BEGIN:STANDARD
DTSTART:16010101T040000
TZOFFSETFROM:+0300
TZOFFSETTO:+0200
RRULE:FREQ=YEARLY;INTERVAL=1;BYDAY=-1SU;BYMONTH=10
END:STANDARD
BEGIN:DAYLIGHT
DTSTART:16010101T030000
TZOFFSETFROM:+0200
TZOFFSETTO:+0300
RRULE:FREQ=YEARLY;INTERVAL=1;BYDAY=-1SU;BYMONTH=3
END:DAYLIGHT
END:VTIMEZONE
BEGIN:VEVENT
ORGANIZER;CN=ExAndroid Developer:mailto:<redacted>@outlook.com
ATTENDEE;ROLE=REQ-PARTICIPANT;PARTSTAT=NEEDS-ACTION;RSVP=TRUE;CN=<redacted>@protonmail.com:mailto:<redacted>@protonmail.com
DESCRIPTION;LANGUAGE=en-US:<stripped>\n\n
UID:040000008200E00074C5B7101A82E00800000000508CC0468E28D701000000000000000
 01000000096DF011F20A29943A70B5DA5047021A5
SUMMARY;LANGUAGE=en-US:Test meeting
DTSTART;TZID=GTB Standard Time:20210403T000000
DTEND;TZID=GTB Standard Time:20210404T000000
CLASS:PUBLIC
PRIORITY:5
DTSTAMP:20210403T103619Z
TRANSP:OPAQUE
STATUS:CONFIRMED
SEQUENCE:0
LOCATION;LANGUAGE=en-US:Microsoft Teams Meeting
X-MICROSOFT-CDO-APPT-SEQUENCE:0
X-MICROSOFT-CDO-OWNERAPPTID:-570210331
X-MICROSOFT-CDO-BUSYSTATUS:TENTATIVE
X-MICROSOFT-CDO-INTENDEDSTATUS:BUSY
X-MICROSOFT-CDO-ALLDAYEVENT:FALSE
X-MICROSOFT-CDO-IMPORTANCE:1
X-MICROSOFT-CDO-INSTTYPE:0
X-MICROSOFT-SKYPETEAMSMEETINGURL:https://teams.microsoft.com/l/meetup-join/
 19%3ameeting_YmM1MjRmMTktYjA2N<stripped>cd8%22%7d
X-MICROSOFT-SCHEDULINGSERVICEUPDATEURL:https://api.scheduler.teams.microsof
 t.com/teams/dc<stripped>DAyMmZj@thread.v2/0
X-MICROSOFT-SKYPETEAMSPROPERTIES:{"cid":"19:meeting_YmM1MjRmMTktYjA2Ny00YWQ
 4LWI1NWEtZmE1NGVlMDAyMmZj@thread.v2"\,"private":true\,"type":0\,"mid":0\,"
 rid":0\,"uid":null}
X-MICROSOFT-ONLINEMEETINGCONFLINK:conf:sip:<redacted>\;gruu\;opaque=
 app:conf:focus:id:teams:2:0!19:meeting_YmM1MjRmMTktYjA2Ny00YWQ4LWI1NWEtZmE
 1NGVlMDAyMmZj-thread.v2!56474ffc245241c5ab4081a127cc1cd8!dcf23acb18fc41d28
 6acf752f1ca658d
X-MICROSOFT-DONOTFORWARDMEETING:FALSE
X-MICROSOFT-DISALLOW-COUNTER:FALSE
X-MICROSOFT-LOCATIONS:[ { "DisplayName" : "Microsoft Teams Meeting"\, "Loca
 tionAnnotation" : ""\, "LocationSource" : 0\, "Unresolved" : false\, "Loca
 tionUri" : "" } ]
BEGIN:VALARM
DESCRIPTION:REMINDER
TRIGGER;RELATED=START:-PT15M
ACTION:DISPLAY
END:VALARM
END:VEVENT
END:VCALENDAR
```

The basic thing to understand here is that every iCalendar object starts with the *"BEGIN:VCALENDAR"* and ends with the *"END:VCALENDAR"*.
The meeting part is between *"BEGIN:VEVENT"* and *"END:VEVENT"*. Inside your event, you specify so-called components. 
Usually, not all of the components are required and in my POC you can find a stripped-down ics file that contains only necessary ones. Most of the components are self-explained but I will mark off the most interesting ones.


**ORGANIZER** - specifies meeting organizer name and email address. You can set any email and name here so it looks like the meeting invite is coming from that person.

**ATTENDEE** - specifies meeting attendee. If you need more attendees you would have multiple ATTENDEE components. Keep in mind attendees won't receive a meeting invite email, this is just for the display. You can easily fake that 30 persons attend the meeting when you actually send an email to a single target.

**X-MICROSOFT-SKYPETEAMSMEETINGURL** - if you specify this component meeting reminder will have the "Join Online" button displayed. Unfortunately, when clicked, it will try to open a specified URL via desktop teams application and fail.

**DTSTART, DTSTAMP, DTEND** - specifies the time of the meeting and how long it lasts. I did a neat trick here and set the meeting start time to 5 minutes before the current time, that way it looks like you are late 5 minutes for the meeting. When the target receives an email, outlook processes it as a meeting invite, it seems that the meeting started 5 minutes ago and immediately displays a reminder on the screen. It helps encourage urgency.

The result looks like this:
![end_result](/assets/end_result.png)

You can check out my POC at [github](https://github.com/ExAndroidDev/fakemeeting/){:target="_blank"} and edit the script and templates to suit your needs.

Keed in mind I've just showed how outlook handles the meeting invites. This should work on any email provider/client that processes ics attachements.

If you have any questions hit me up on twiter at [@ExAndroidDev](https://twitter.com/ExAndroidDev){:target="_blank"}.

Then end!
