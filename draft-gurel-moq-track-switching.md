---
title: "Track Switching in Media over QUIC Transport"
abbrev: moq-track-switching
docname: draft-gurel-moq-track-switching-latest
date: {DATE}
category: std

ipr: trust200902
area: Applications and Real-Time
submissionType: IETF
workgroup: MOQ
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs, docmapping]

author:
  -
    ins: Z. Gurel
    name: Zafer Gurel
    organization: Ozyegin University
    email: zafer.gurel@ozu.edu.tr

  -
    ins: A. Begen
    name: Ali Begen
    organization: Networked Media
    email: ali.begen@networked.media

normative:
  MoQTransport: I-D.ietf-moq-transport
  CommonCatalogFormat: I-D.ietf-moq-catalogformat

informative:

--- abstract

This document defines a solution for switching tracks in media.
More particularly, the solution provides a seamless switching that
ensures there is no overlapping or gap between the download and/or
transmission of two tracks when they are alternatives to each other.

--- middle

# Introduction

This document outlines a solution for switching tracks in media
delivery over Media Over QUIC Transport (MOQT) {{MoQTransport}}. Switching tracks
is necessary for a variety of reasons, such as changing the
quality, the language of the media, or the type of the media (e.g., switching from a video to
an audio track). The solution described in this document ensures
that there is no overlapping or gap between the download and/or
transmission of two tracks when they are alternatives to each other.

## Terms and Definitions

{::boilerplate bcp14-tagged}

Client:

: The party initiating a MoQ transport session.

Server:

: The party accepting an incoming transport session.

Endpoint:

: A Client or Server.

Producer:

: An endpoint sending media over the network.

Consumer:

: An endpoint receiving media over the network.

Transport session:

: A raw QUIC connection or a WebTransport session.

Congestion:

: Packet loss and queuing caused by degraded or overloaded networks.

Group:

: A temporal sequence of objects. A group represents a join point in a
  track.

Object:

: An object is an addressable unit whose payload is a sequence of
  bytes.

Track:

: An encoded bitstream. Tracks contain a sequential series of one or
  more groups and are the subscribable entity with MOQT.

## Notational Conventions

This document uses the conventions detailed in ({{?RFC9000, Section 1.3}})
when describing the binary encoding.

As a quick reference, the following list provides a non-normative summary
of the parts of RFC9000 field syntax that are used in this specification.

x (L):

: Indicates that x is L bits long

x (i):

: Indicates that x holds an integer value using the variable-length
  encoding as described in ({{?RFC9000, Section 16}})

x (..):

: Indicates that x can be any length including zero bits long.  Values
 in this format always end on a byte boundary.

x (L) ...:

: Indicates that x is repeated zero or more times and that each instance
  has a length of L

This document extends the RFC9000 syntax and with the additional field types:

x (b):

: Indicates that x consists of a variable length integer encoding as
  described in ({{?RFC9000, Section 16}}), followed by that many bytes
  of binary data

To reduce unnecessary use of bandwidth, variable length integers SHOULD
be encoded using the least number of bytes possible to represent the
required value.

# Switching Tracks

In MOQT communications, the publisher announces the availability
of multiple encodings of a media content in different tracks,
which are alternatives of each other and indicated so in the catalog {{CommonCatalogFormat}}.
The subscriber subscribes to one of the tracks from an altGroup
in the catalog. During the session, the subscriber may switch from
a currently consumed track to any other alternate track from the
catalog due to, for example, changes in available bandwidth. To do this,
the subscriber can subscribe to a new track and unsubscribe from the old track.
Such an action is done by sending a SUBSCRIBE message to the relay.
An example of the different tracks indicated in the catalog is shown below.

~~~
{
    "tracks":[
        {
          "name": "hd",
          "selectionParams": {
            "width": 1920, "height": 1080,
            "bitrate": 5000000, "framerate": 30
          },
          "altGroup": 1
        },
        {
          "name": "md",
          "selectionParams": {
            "width": 720, "height": 640,
            "bitrate": 3000000, "framerate": 30
          },
          "altGroup": 1
        },
        {
          "name": "sd",
          "selectionParams": {
            "width": 192, "height": 144,
            "bitrate": 500000, "framerate": 30
          },
          "altGroup": 1
        },
        {
          "name": "audio",
          "selectionParams": {
            "codec": "opus", "samplerate": 48000,
            "channelConfig": "2", "bitrate": 32000
          }
        }
      }
    ]
}
~~~
{: #moq-transport-catalog-snippet title="An example of the different tracks."}

## The Problem and Solution Approaches
Relays do not have access/visibility to the catalog. Therefore, they are unaware when two tracks are alternates. An example of the existing SUBSCRIBE message format is shown below.

~~~
SUBSCRIBE Message {
  Subscribe ID (i),
  Track Alias (i),
  Track Namespace (b),
  Track Name (b),
  Filter Type (i),
  [StartGroup (i),
   StartObject (i)],
  [EndGroup (i),
   EndObject (i)],
  Number of Parameters (i),
  Subscribe Parameters (..) ...
}
~~~
{: #moq-transport-subscribe-format title="MOQT SUBSCRIBE message."}

Existing SUBSCRIBE message that the subscriber transmits to the relay only contains information of the current track and does not indicate that the client is switching to a new track for the same media content. Therefore, when receiving a SUBSCRIBE message from the subscriber for switching to the new track, the relay may download and transmit both the new track and the old track of the same media content, which can create a bitrate spike and in turn can aggravate an already congested link. Additionally, the player/client application on the subscriber will have to process (e.g., parse and decode) the same media content in overlapping times, which is a waste of computational power.

### Solution 1 (altTrackGroup)

A new parameter altTrackGroup can be added to every SUBSCRIBE message. altTrackGroup is the identifier for a group of alternative tracks within the scope of a track namespace. The value of the altTrackGroup identifier may be the same as the altGroup identifier used in the catalog or a different one. An example of a SUBSCRIBE message that includes the identifier altTrackGroup is shown below.

~~~
SUBSCRIBE Message {
  Subscribe ID (i),
  Track Alias (i),
  Track Namespace (b),
  Track Name (b),
  altTrackGroup (i),
  Filter Type (i),
  [StartGroup (i),
   StartObject (i)],
  [EndGroup (i),
   EndObject (i)],
  Number of Parameters (i),
  Subscribe Parameters (..) ...
}
~~~
{: #moq-transport-subscribe-format-atg title="MOQT SUBSCRIBE message with altTrackGroup."}

### Solution 2 (Switch Track Alias)

The SUBSCRIBE message can contain an identifier Switch Track Alias such that the Switch Track Alias = Track Alias of the active subscription. This way, this ID in the SUBSCRIBE message can indicate to the relay that this switching request is for an alternative track of the same media content of the current track and assists the relay in seamless switching.  An example of a SUBSCRIBE message that includes the identifier Switch Track Alias is shown below.

~~~
SUBSCRIBE Message {
  Subscribe ID (i),
  Track Alias (i),
  Track Namespace (b),
  Track Name (b),
  Switch Track Alias (i),
  Filter Type (i),
  [StartGroup (i),
   StartObject (i)],
  [EndGroup (i),
   EndObject (i)],
  Number of Parameters (i),
  Subscribe Parameters (..) ...
}
~~~
{: #moq-transport-subscribe-format-sta title="MOQT SUBSCRIBE message with Switch Track Alias."}
