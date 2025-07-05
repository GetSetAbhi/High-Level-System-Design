# Dropbox / Google Drive

Took references from **System Design Interview by Alex Xu Volume I**, A talk by a [dropbox engineer](https://www.youtube.com/watch?v=PE4gwstWhmc)

## File Upload Workflow

1. Client-Side File Preparation (User Device):

	* The User selects a file for upload.
	* The User Device application chunks the file into smaller, fixed-size blocks (e.g., 4MB).
	* For each block, the User Device computes a cryptographic hash (e.g., SHA-256) of its content.
2. Initiate Upload Session & Request Block Status (User Device to File Service via REST, then File Service to Block Service via RPC):

	* The User Device sends an initial POST */files/upload/init REST request to the File Service, including the file's logical metadata (filename, total size, parent folder ID) and a list of all block hashes and their sizes.
	* The File Service receives this, then makes an RPC call (e.g., BlockService.GetOrCreateBlockLocations) to the Block Service, passing the list of block hashes and sizes.

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
	to video into different audio and video segments etc., storing them in s3.

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