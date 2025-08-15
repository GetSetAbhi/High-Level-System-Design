# TCP and UDP

Most of the client server communication happens over TCP (Transmission Control Protocol) using HTTP, HTTPS or Websocket.

TCP is a connection oriented protocol, to establish 1 connection between client and a server, 3 way handshake occurs between client and server.
TCP is a lossless protocol and it ensures data packets are delivered in order, without duplication, and retransmits any lost packets. 
This reliability introduces overhead compared to UDP, which does not guarantee delivery or ordering.

UDP is a connectionless protocol and doesn't guarantee ordering of data. Also some data packets might be lost during transmission. UDP ensures at most once delivery of messages.

**Why UDP is useful for Video Calling / Live Streaming**

* Low Latency – In real-time applications, it’s better to drop a frame than wait for it to be resent. TCP retransmissions would cause delays and freeze the video.
* Continuous Flow – Even if some packets are lost, the video/audio can keep playing without waiting.
* No Ordering Requirement – A lost or late video frame isn’t useful anymore; UDP just sends the next one without delay.
* Less Overhead – Smaller headers and no connection management mean more bandwidth for actual audio/video data.


## 1 - 1 Video calling Scenario

![1-1 Video Calling](p2p.svg)

For a one-to-one video call between User 1 (U1) and User 2 (U2), both devices need to establish a direct connection over *UDP (User Datagram Protocol)* for efficient media transfer. To achieve this, U1 and U2 first connect to a *STUN (Session Traversal Utilities for NAT)* server. The *STUN* server's role is to help each device discover its public IP address and port, a process known as "NAT traversal."

This public address information is then exchanged between U1 and U2 through a signaling channel, typically a WebSocket. This channel is also used to negotiate technical details, such as supported video codecs, resolutions, and available bandwidth. This negotiation process, using the *Session Description Protocol (SDP)*, ensures that the call quality is optimized for the lowest-capability device or network. For example, if U2 is on a low-bandwidth connection, both devices will agree to transmit a lower-resolution video stream.

After the exchange and negotiation are complete, the devices attempt to establish a direct, peer-to-peer UDP connection using the public IP addresses they discovered. The actual video call is then a continuous stream of data packets flowing directly between the two devices, with each acting as both a sender and a receiver.

In scenarios where a direct peer-to-peer connection is blocked by a restrictive firewall or network configuration (such as a symmetric NAT), the connection attempt will fail. In such cases, the system falls back to a *TURN (Traversal Using Relays around NAT)* server. The *TURN* server does not just help with finding IP addresses; its primary function is to relay all the UDP media traffic between U1 and U2. This ensures the call can still proceed, though with slightly more latency due to the data being routed through an intermediary server.

A firewall interruption occurs at the edge of the local network, such as a home router's NAT firewall or a corporate network's gateway. These firewalls are designed to block incoming connections that aren't a direct response to an outgoing request. If this interruption prevents a direct UDP connection, the system falls back to a TURN (Traversal Using Relays around NAT) server. The TURN server's role is to relay all the UDP media traffic between U1 and U2, ensuring the call can still proceed, though with slightly more latency due to the data being routed through a middleman server.

All of this is part of the overall WebRTC framework.


## Video Post-Processing/ Video Transcoding Service

![Video Post-Processing Service](video-transcoding.svg)

Whenever we upload a video, it is stored into an Object Storage like S3.
Once a file is stored in S3, S3 triggers an event and sends video to Video Transcoding Service.

The post-processing/Video Transcoding Service is a "pipeline" which produces the following outputs:

1. Video segment files in different formats (codec and container combinations) stored in S3.
	
2. Manifest files (a primary manifest file and several media manifest files) stored in S3. 
The media manifest files will reference segment files in S3.


The Video Post Processing Pipeline can be thought of as a DAG, which embodies the properties of a DAG:

* Directed: Each step in the pipeline has a clear, one-way flow. For example, you must split a video into 
segments before you can transcode those segments. Data moves forward from one task to the next; it never flows backward.

* Acyclic: There are no "loops" or "cycles" in the workflow. A task will never depend on itself, 
nor will a series of tasks eventually lead back to a task that has already been completed. 
This ensures the pipeline always progresses towards completion and avoids infinite processing loops.

**Internal Working of Video Post-Processing Service**

1. Whenever a video is uploaded completely into S3, S3 Event Notification triggers an AWS Lambda function 
which starts an AWS Step Function Workflow. AWS Step Functions is designed precisely to orchestrate and encompass 
Lambda functions and MediaConvert jobs into a cohesive workflow.

2. The Step Functions workflow performs these steps:

	* (Segmentation): Invokes another Lambda function (or a Fargate task via AWS Batch) 
	to partiyion video into different audio and video segments etc., storing them in s3.

	* Video and Audio Encoding: These are lambdas, that are orchestrated by the AWS Step Function workflow 
	and these parallely encode audio and video segments into various resolutions and bitrates 
	to support adaptive bitrate streaming. The results of these processes are stored in S3, so that they can be assembled later.

	* Thumbnail Generation: This is another lambda function that generates a thumbnail for the video.

	* Assembly and Manifest Generation: Once Audio Encoding and Video Encoding for various segments are done, 
	the AWS Step Function workflow then triggers another lambda that assembles the various audio and video encodings 
	that were generated in previous steps and generates a manifest file which acts as an index of these encodings. 
	manifest file is then stored in the S3.

	* The last step of this process would be to update the metadata db containing the meta data details if the video.

**Choice of Database for storing Metadata**

For a system like YouTube or Spotify, you would almost certainly use a combination of these databases (a polyglot persistence approach):

* Relational Database (e.g., PostgreSQL/MySQL): For core, highly structured metadata like video/audio IDs, titles, upload dates, user account information, 
and critical relationships that require strong consistency (e.g., which user owns which video).

* Document Database (e.g., MongoDB): For more flexible, evolving metadata like tags, specific video properties, or user-generated content details that don't fit neatly into a rigid schema.
	
---
# Audio Streaming Service

![Audio Post-Processing Service](audio_transcoding.svg)

The high level design for an Audio Post-Processing and Transcoding service remains same.
We just remove video specific parts but rest of the things remain the same.