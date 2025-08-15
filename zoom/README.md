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