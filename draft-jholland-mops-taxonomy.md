---
title: Operational Considerations for Streaming Media
abbrev: Media Streaming Ops
docname: draft-jholland-mops-taxonomy-01
date: 2020-01-17
category: info

ipr: trust200902
area: Ops
workgroup: Mops
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: J. Holland
    name: Jake Holland
    org: Akamai Technologies, Inc.
    street: 150 Broadway
    city: Cambridge, MA 02144
    country: United States of America
    email: jakeholland.net@gmail.com

informative:
  CVNI:
    target: https://www.cisco.com/c/en/us/solutions/collateral/service-provider/visual-networking-index-vni/white-paper-c11-741490.html
    title: "Cisco Visual Networking Index: Forecast and Trends, 2017–2022 White Paper"
    author:
      - 
        org: "Cisco Systems, Inc."
    date: 2019-02-27
  DASH:
    title: "Information technology -- Dynamic adaptive streaming over HTTP (DASH) -- Part 1: Media presentation description and segment formats"
    seriesinfo:
      "ISO/IEC": 23009-1:2014
    date: 2014-05
  MSOD:
    title: "Media Services On Demand: Encoder Best Practices"
    author:
      - 
        org: "Akamai Technologies, Inc."
    target: https://learn.akamai.com/en-us/webhelp/media-services-on-demand/media-services-on-demand-encoder-best-practices/GUID-7448548A-A96F-4D03-9E2D-4A4BBB6EC071.html
    date: 2019
  RFC2309:
  RFC3168:
  RFC5762:
  RFC6190:
  RFC8033:
  RFC8216:
  I-D.han-iccrg-arvr-transport-problem:

--- abstract

This document provides an overview of operational networking issues
that pertain to quality of experience in delivery of video and other
high-bitrate media over the internet.

--- middle

#Introduction {#intro}

As the internet has grown, an increasingly large share of the traffic
delivered to end users has become video.  Estimates
put the total share of internet video traffic at 75% in 2019, expected
to grow to 82% by 2022.  What's more, this estimate projects the
gross volume of video traffic will more than double during this time,
based on a compound annual growth rate continuing at 34% (from Appendix
D of {{CVNI}}).

In many contexts, video traffic can be handled transparently as
generic application-level traffic.  However, as the volume of
video traffic continues to grow, it's becoming increasingly
important to consider the effects of network design decisions
on application-level performance, with considerations for
the impact on video delivery.

This document aims to provide a taxonomy of networking issues as
they relate to quality of experience in internet video delivery.
The focus is on capturing characteristics of video delivery that
have surprised network designers or transport experts without
specific video expertise, since these highlight key differences
between common assumptions in existing networking documents and
observations of video delivery issues in practice.

Making specific recommendations for mitigating these issues
is out of scope, though some existing mitigations are mentioned
in passing.  The intent is to provide a point of reference for
future solution proposals to use in describing how new
technologies address or avoid these existing observed problems.

#Bandwidth Provisioning

##Scaling Requirements for Media Delivery {#scaling}

###Video Bitrates

Video bit-rate selection depends on many variables.  Different
providers give different guidelines, but an equation that
approximately matches the bandwidth requirement estimates
from several video providers is given in {{MSOD}}:

~~~
Kbps = (HEIGHT * WIDTH * FRAME_RATE) / (7 * 1024)
~~~

Height and width are in pixels, and frame rate in frames per second.
The actual bit-rate required for a specific video will also depend on the
codec used and some other characteristics of the video itself, such
as the frequency of high-detail motion, which may influence the
compressability of the content, but this equation provides a rough
estimate.

Here are a few common resolutions used for video content, with their
minimum per-user bandwidth requirements according to this formula:

| Name | Width x Height | Approximate Bit-rate for 60fps
| -----+----------------+-------------------------------
| DVD |  720 x 480 | 3 Mbps
| 720p | 1280 x 720 | 8 Mbps
| 1080p | 1920 x 1080 | 18 Mbps
| 2160p (4k) | 3840 x 2160 | 70 Mbps

###Virtual Reality Bitrates

TBD: Reference and/or adapt content from expired
work-in-progress {{I-D.han-iccrg-arvr-transport-problem}}.

The punchline is that it starts at a bare minimum of 22 Mbps
mean with a 130 Mbps peak rate, up to 3.3 Gbps mean with 38 Gbps
peak for high-end technology.

##Path Requirements

The bit-rate requirements in {{scaling}} are per end-user actively
consuming a media feed, so in the worst case, the bit-rate demands
can be multiplied by the number of simultaneous users to find the
bandwidth requirements for a router on the delivery path with that
number of users downstream.  For example, at a node with 10,000
downstream users simultaneously consuming video streams,
approximately up to 180 Gbps would be necessary in order for all
of them to get 1080p resolution at 60 fps.

However, when there is some overlap in the feeds being consumed by
end users, it is sometimes possible to reduce the bandwidth
provisioning requirements for the network by performing some kind
of replication within the network.  This can be achieved via object
caching with delivery of replicated objects over individual
connections, and/or by packet-level replication using multicast.

To the extent that replication of popular content can be performed,
bandwidth requirements at peering or ingest points can be reduced to
as low as a per-feed requirement instead of a per-user requirement.

##Caching Systems

TBD: pros, cons, tradeoffs of caching designs at different locations within
the network?

Peak vs. average provisioning, and effects on peering point congestion
under peak load?

Provisioning issues for caching systems?

##Predictable Usage Profiles

TBD: insert charts showing historical relative data usage patterns with
error bars by time of day in consumer networks?

Cross-ref vs. video quality by time of day in practice for some case
study?  Not sure if there's a good way to capture a generalized insight
here, but it seems worth making the point that demand projections can
be used to help with e.g. power consumption with routing architectures
that provide for modular scalability.

#Adaptive Bit Rate

##Overview

Adaptive Bit-Rate (ABR) is a sort of application-level congestion
response strategy in which the receiving media player attempts to
detect the available bandwidth of the network path by experiment
or by observing the successful application-layer download speed,
then chooses a video bitrate that fits within that bandwidth,
typically adjusting as changes in available bandwidth occur in
the network.

The choice of bit-rate occurs within the context of optimizing for
some metric monitored by the video player, such as highest achievable
video quality, or lowest rate of expected rebuffering events.

##Segmented Delivery

ABR strategies are commonly implemented by video players using HLS
{{RFC8216}} or DASH {{DASH}} to perform a reliable segment delivery
of video data over HTTP.  Different player implementations and
receiving devices use different strategies, often proprietary
algorithms, to perform the bit-rate selection and available
bandwidth estimation.

This kind of bandwidth-detection system can experience trouble in
several ways that can be affected by networking design choices.

###Idle Time Between Segments

When the bit-rate selection is successfully chosen below the
available capacity of the network path, the response to a
segment request will complete in less absolute time than the
video bit-rate speed.

The resulting idle time within the connection carrying the
segments has a few surprising consequences:

 * Mobile flow-bandwidth spectrum and timing mapping.

 * TCP Slow-start when restarting after idle requires multiple
   RTTs to re-establish a throughput at the network's available
   capacity.  On high-RTT paths or with small enough segments,
   this can produce a falsely low application-visible measurement
   of the available network capacity.

###Head of Line Blocking

In the event of a lost packet on a TCP connection with SACK
support (a common case for segmented delivery in practice), loss
of a packet can provide a confusing bandwidth signal to the
receiving application.  Because of the sliding window in TCP,
many packets may be accepted by the receiver without being available
to the application until the missing packet arrives.  Upon arrival
of the one missing packet after retransmit, the receiver will
suddenly get access to a lot of data at the same time.

To a receiver measuring bytes received per unit time at the
application layer, and interpreting it as an estimate of the
available network bandwidth, this appears as a high jitter in
the goodput measurement.

Active Queue Management (AQM) systems such as PIE {{RFC8033}} or
variants of RED {{RFC2309}}} that induce early random loss under
congestion can mitigate this by using ECN {{RFC3168}} where
available.  ECN provides a congestion signal and induce a similar
backoff in flows that use Explicit Congestion Notification-capable
transport, but by avoiding loss avoids inducing head-of-line blocking
effects in TCP connections.

##Unreliable Transport

In contrast to segmented delivery, several applications use UDP or
unreliable SCTP to deliver RTP or raw TS-formatted video.

Under congestion and loss, this approach generally experiences more
video artifacts with fewer delay or head of line blocking effects.
Often one of the key goals is to reduce latency, to better support
applications like video conferencing, or for other live-action
video with interactive components, such as some sporting events.

Congestion avoidance strategies for this kind of deployment vary
widely in practice, ranging from some streams that are entirely
unresponsive to using feedback signaling to change encoder settings
(as in {{RFC5762}}), or to use fewer enhancement layers (as in
{{RFC6190}}), to proprietary methods for detecting quality of
experience issues and cutting off video.

# Doc History and Side Notes

Note to RFC Editor: Please remove this section before publication

TBD: suggestion from mic at IETF 106 (Mark Nottingham): dive into
the different constraints coming from different parts of the network
or distribution channels. (regarding questions about how to describe
the disconnect between demand vs. capacity, while keeping good archival
value.)
https://www.youtube.com/watch?v=4_k340xT2jM&t=13m

TBD: suggestion from mic at IETF 106 (Dave Oran + Glenn Deen responding):
pre-placement for many use cases is useful--distinguish between live vs.
cacheable.  "People assume high-demand == live, but not always true" with
popular netflix example.

(Glenn): something about latency requirements for cached vs. streaming on
live vs.  pre-recorded content, and breaking requirements into 2 separate
charts.  also: "Standardized ladder" for adaptive bit rate rates suggested,
declined as out of scope.
https://www.youtube.com/watch?v=4_k340xT2jM&t=14m15s

TBD: suggestion at the mic from IETF 106 (Aaron Falk): include
industry standard metrics from citations, some standard scoping metrics
may be already defined.
https://www.youtube.com/watch?v=4_k340xT2jM&t=19m15s

#IANA Considerations

This document requires no actions from IANA.

#Security Considerations

This document introduces no new security issues.

--- back

