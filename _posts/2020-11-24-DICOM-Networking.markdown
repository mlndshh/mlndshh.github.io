---
layout: post
title:  "DICOM Networking"
date:   2020-11-24 15:10:50 +0530
categories: dicom
---
## DICOM Network protocol

The DICOM Protocol predates SOAP and REST by quite a lot. One can do four things with it:

  

-  **Test the connection** between two devices (C-Echo).

-  **Send images** from the local imaging device to a remote device (C-Store).

-  **Search the content** of a remote device (C-Find).

-  **Retrieve images** from a remote device (C-Move or C-Get).

  

![](/images/dicom/scu-scp.png)

  

The client of a DICOM service is known as a **service class user** (SCU), and the server that handles the requests is called a **service class provider** (SCP). The client sends a request that is encoded as a DICOM file (the **command**), and the server answers with a DICOM file.

  

The connection between a server and client is called an association. An association starts with a handshake where both parties agree on which commands can be exchanged between them and which transfer syntaxes will be supported. The result of this negotiation is called the **presentation context**. Once the association is negotiated, this communication channel can be used to successively send multiple, independent commands.

  

### Parameters of a DICOM server

A DICOM server can be identified by:

  

- Its **IP address** (or, equivalently, its symbolic DNS hostname).

- Its **TCP port** (the standard DICOM port is 104, but Orthanc uses the non-priviliged port 4242 by default).

  

Along with the above, each imaging device (client or server) must be associated with a symbolic name that is called the **application entity title (AET)**. The AET is assumed to be unique inside the Intranet of the hospital. For best compatibility between vendors, the AET should be only made of alphanumeric characters in upper case (plus the “`-`” and “`_`” characters), and its length must be below 16 characters.

  

So the IP Address, TCP Port and the AET describe all the parameters of a DICOM server.

  

### Commands

-  **C-Echo** - C-Echo is used to test client-server to test DICOM-level connectivity.

-  **C-Store** - C-Store is used to send DICOM instances to a remote imaging device

-  **C-Find** - C-Find is used to search a list of DICOM resources that are hosted by some remote DICOM server. The kind of resource that is needed patients, studies or series) must be specified. The query also contains a set of filters on DICOM tags of interest.

-  **C-Move** - C-Move is notably used to locally retrieve DICOM files from a remote server, given the results of a C-Find query.

-  **C-Get** - C-Get provides a simpler alternative to DICOM C-Move, if the _issuer_ and the _target_ correspond to the same modality

  

### Dicom Modalities

Modality is used in radiology to refer to one form of **imaging**. A DICOM Modality will give information about the type of imaging in the specified DICOM.

### Picture archiving and communication system (PACS)

A PACS is a medical imaging technology used to store and access images from multiple modalities. PACS was introduced to essentially eliminate the need to physically and manually file, retrieve, or transport imagery. There. are four components of PACS:

- **Imaging Modalities** - This is the image system for doing the actual scanning of a patient in producing a medical image (eg. CT, MRI, PF).
- A **secured network** for the transmission of patient information
- **Workstations** for interpreting and reviewing images
- **Archives** for the storage and retrieval of images and reports

Communication with PACS servers is done through DICOM commands (like the ones listed above). To understand how the communication works, we will be making use of:

 - Orthanc - Orthanc is an open-source DICOM server for medical imaging. It acts as a mini-PACS system. You can get it from [here](https://www.orthanc-server.com/download.php).
 - DCMTK - DCMTK is a collection of libraries and applications implementing large parts the DICOM standard. DCMTK is incredibly expansive, and we will only be using it to run DICOM commands from the terminal.

To run the Orthanc server, I used Docker Compose. Below is what my `docker-compose.yml` looks like:

```
version: "3"
services:
  pacs:
    image: jodogne/orthanc-plugins
    ports:
      - 8042:8042
      - 4242:4242
    volumes:
      - ./orthanc.json:/etc/orthanc/orthanc.json:ro
      - orthanc_db:/var/lib/orthanc/db/
    expose: 
      - 4242

volumes: 
  orthanc_db:
```

The first volume is for the orthanc.json file, the contents of which you can find here (link to json file).

Let's take each command one by one and use dcmtk to communicate to our orthanc server.

### **C-ECHO** - [echoscu](https://support.dcmtk.org/docs/echoscu.html)

`echoscu [options] peer port`

echoscu needs 2 arguments to work:

- peer - hostname of DICOM peer (ip)
- port - tcp/ip port number of peer

Since you are running the orthanc server locally (you're technically running it inside the docker container, but we are using port forwarding for port 4242), `peer` will be `127.0.0.1` (or `localhost` or `0.0.0.0`). For the port, you will have to look at the config JSON file that is used with your Orthanc server and look for `DicomPort`. The default value is 4242.

Ensure that the Orthanc server is running, and we can run the command

`echoscu localhost 4242`

```
	(base) milind@Milinds-MacBook-Air ohif % echoscu localhost 4242 
	(base) milind@Milinds-MacBook-Air ohif % 
```
	
There was no output, let's add the `-v` flag and see what happens

	(base) milind@Milinds-MacBook-Air ohif % echoscu -v localhost 4242
	I: Requesting Association
	I: Association Accepted (Max Send PDV: 16372)
	I: Sending Echo Request (MsgID 1)
	I: Received Echo Response (Success)
	I: Releasing Association

That's a good start! We're able to communicate to our orthanc server without issues. Let's move onto actually transferring data now.

### **C-STORE** - [storescu](https://support.dcmtk.org/docs/storescu.html) 
Sending a DICOM to the orthanc server

`storescu [options] peer port dcmfile-in`

`dcmfile-in` is the path of the dicom file you want to send. Let's try this out.

![](/images/dicom-networking/cstorebefore.png)

	(base) milind@Milinds-MacBook-Air ohif % storescu localhost 4242 demo1.dcm
	W: DIMSE Warning: (STORESCU,ANY-SCP): sendMessage: unable to convert dataset from 'JPEG Lossless, Non-hierarchical, 1st Order Prediction' transfer syntax to 'Little Endian Explicit'
	E: Store Failed, file: demo1.dcm:
	E: 0006:020e DIMSE Failed to send message
	E: Store SCU Failed: 0006:020e DIMSE Failed to send message
	(base) milind@Milinds-MacBook-Air ohif % 

Well, we get an error saying it cannot dataset from 'JPEG Lossless, Non-hierarchical, 1st Order Prediction' transfer syntax to 'Little Endian Explicit'. To fix this, let's add the `-xs` flag. From the docs, `-xs`is used to `propose default JPEG lossless TS and all uncompressed transfer syntaxes`. Let's try again.

	(base) milind@Milinds-MacBook-Air ohif % storescu -xs localhost 4242 demo1.dcm
	(base) milind@Milinds-MacBook-Air ohif % 

No errors now! Let's check Orthanc again

![](/images/dicom-networking/cstoreafter.png)

There it is, we have used C-STORE to send a dicom file to our Orthanc server! You can use `-v` here too.

	(base) milind@Milinds-MacBook-Air ohif % storescu -xs -v localhost 4242 demo1.dcm
	I: checking input files ...
	I: Requesting Association
	I: Association Accepted (Max Send PDV: 16372)
	I: Sending file: demo1.dcm
	I: Converting transfer syntax: JPEG Lossless, Non-hierarchical, 1st Order Prediction -> JPEG Lossless, Non-hierarchical, 1st Order Prediction
	I: Sending Store Request (MsgID 1, CR)
	XMIT: ..................................................................................................................................................................................................
	I: Received Store Response (Success)
	I: Releasing Association


### **C-FIND** - [findscu](https://support.dcmtk.org/docs/findscu.html) 
Sending a query to find DICOMs

This is where things get interesting. Look art this [findscu example](https://forum.dcmtk.org/viewtopic.php?t=116) from the dcmtk FAQs

```
findscu -v -P -k 0008,0052="IMAGE" -k 0010,0020="300019" -k 0020,000D="1.2.3.1" -k 0020,000E="1.2.3.2" -k 0008,0018="1.2.3.3" localhost 104
```

We will have to use the hex values for the DICOM tags!
`-P` is to use patient root information model
`-k` is to look for values according to the dicom tags. `0010,0020` is the patient ID for example. (`0010,0010` is patient name). You can also use dictionary names instead of the hex values

With this information, let's try this ourselves. Below is the study I'll try to find.

![](/images/dicom-networking/abcdpatient.png)

Let's run this simple command: `findscu -v -P -k "(0008,0052)=PATIENT" -k PatientID="CLU121161" localhost 4242`

```
(base) milind@Milinds-MacBook-Air ohif % findscu -v -P -k "(0008,0052)=PATIENT" -k PatientID="CLU121161" localhost 4242
I: Requesting Association
I: Association Accepted (Max Send PDV: 16372)
I: Sending Find Request (MsgID 1)
I: Request Identifiers:
I: 
I: # Dicom-Data-Set
I: # Used TransferSyntax: Little Endian Explicit
I: (0008,0052) CS [PATIENT]                                #   8, 1 QueryRetrieveLevel
I: (0010,0020) LO [CLU121161]                              #  10, 1 PatientID
I: 
E: Find Failed, query keys:
E: 
E: # Dicom-File-Format
E: 
E: # Dicom-Meta-Information-Header
E: # Used TransferSyntax: Little Endian Explicit
E: 
E: # Dicom-Data-Set
E: # Used TransferSyntax: Little Endian Explicit
E: (0008,0052) CS [PATIENT ]                               #   8, 1 QueryRetrieveLevel
E: (0010,0020) LO [CLU121161 ]                             #  10, 1 PatientID
E: 
E: 0006:0317 Peer aborted Association (or never connected)
I: Peer Aborted Association
```

Hmm. It says peer aborted association? Let's look at what the Orthanc server is saying.

![](/images/dicom-networking/orthancimage.png)

`Rejected Find request from remote DICOM modality with AET "FINDSCU" and hostname "172.29.0.1"`

It's rejecting our find request. This is because it needs the dicom modality info in its config file. In `"DicomModalities"`, based on my IP that it receives (it's not local because I'm running orthanc in a docker container), I am adding: 
`"test" : ["FINDSCU", "172.29.0.1", 2000],`

It now knows to accept requests from AETitle `FINDSCU` (which is the default title used when findscu is used) with the given IP address and port. Let's restart the server and try again.

```
(base) milind@Milinds-MacBook-Air ohif % findscu -v -P -k "(0008,0052)=PATIENT" -k PatientID="CLU121161" localhost 4242
I: Requesting Association
I: Association Accepted (Max Send PDV: 16372)
I: Sending Find Request (MsgID 1)
I: Request Identifiers:
I: 
I: # Dicom-Data-Set
I: # Used TransferSyntax: Little Endian Explicit
I: (0008,0052) CS [PATIENT]                                #   8, 1 QueryRetrieveLevel
I: (0010,0020) LO [CLU121161]                              #  10, 1 PatientID
I: 
I: ---------------------------
I: Find Response: 1 (Pending)
I: 
I: # Dicom-Data-Set
I: # Used TransferSyntax: Little Endian Explicit
I: (0008,0005) CS [ISO_IR 100]                             #  10, 1 SpecificCharacterSet
I: (0008,0052) CS [PATIENT ]                               #   8, 1 QueryRetrieveLevel
I: (0008,0054) AE [ORTHANC ]                               #   8, 1 RetrieveAETitle
I: (0010,0020) LO [CLU121161 ]                             #  10, 1 PatientID
I: 
I: Received Final Find Response (Success)
I: Releasing Association
```

It works! `0008,0052` is the `Query/Retrieve Level` and needs to be specified each time. There are 4 possibilities (PATIENT, STUDY, SERIES or IMAGE) and it is used to target what level we would like to query at. In the example above, I am querying at patient level, looking for `PatientID="CLU121161`. While my query was a success, the information retrieved is not of much help. Let's change this.

```
(base) milind@Milinds-MacBook-Air ohif % findscu -P -k "(0008,0052)=PATIENT" -k PatientID="CLU121161" -k PatientName localhost 4242 
I: ---------------------------
I: Find Response: 1 (Pending)
I: 
I: # Dicom-Data-Set
I: # Used TransferSyntax: Little Endian Explicit
I: (0008,0005) CS [ISO_IR 100]                             #  10, 1 SpecificCharacterSet
I: (0008,0052) CS [PATIENT ]                               #   8, 1 QueryRetrieveLevel
I: (0008,0054) AE [ORTHANC ]                               #   8, 1 RetrieveAETitle
I: (0010,0010) PN [ABCD]                                   #   4, 1 PatientName
I: (0010,0020) LO [CLU121161 ]                             #  10, 1 PatientID
```

This is already better! I added `-k PatientName`, to find out what the Name of the patient with `PatientID="CLU121161"` is. Let's take it a step further:

```
(base) milind@Milinds-MacBook-Air ohif % findscu -v -P -k "(0008,0052)=PATIENT" -k PatientID -k PatientName="A*" localhost 4242
I: Requesting Association
I: Association Accepted (Max Send PDV: 16372)
I: Sending Find Request (MsgID 1)
I: Request Identifiers:
I: 
I: # Dicom-Data-Set
I: # Used TransferSyntax: Little Endian Explicit
I: (0008,0052) CS [PATIENT]                                #   8, 1 QueryRetrieveLevel
I: (0010,0010) PN [A*]                                     #   2, 1 PatientName
I: (0010,0020) LO (no value available)                     #   0, 0 PatientID
I: 
I: ---------------------------
I: Find Response: 1 (Pending)
I: 
I: # Dicom-Data-Set
I: # Used TransferSyntax: Little Endian Explicit
I: (0008,0005) CS [ISO_IR 100]                             #  10, 1 SpecificCharacterSet
I: (0008,0052) CS [PATIENT ]                               #   8, 1 QueryRetrieveLevel
I: (0008,0054) AE [ORTHANC ]                               #   8, 1 RetrieveAETitle
I: (0010,0010) PN [ABCD]                                   #   4, 1 PatientName
I: (0010,0020) LO [CLU121161 ]                             #  10, 1 PatientID
I: 
I: Received Final Find Response (Success)
I: Releasing Association
```

This is sweet! If you didn't follow the command, I'm looking for patients with names starting with `A`, and getting the PatientID in return! Here's another example

```
(base) milind@Milinds-MacBook-Air ohif % findscu -v -P -k 0008,0052="IMAGE" -k 0010,0020="CLU121161" -k 0020,000D="1.2.840.10008.1.500817.567467863.1560288934.406684597" -k 0020,000E="1.2.392.200036.9125.3.331555024156.64663542911.19447643" -k 0008,0018="1.2.392.200036.9125.9.0.235875605.654909696.849058840" -k Modality -k PerformedProcedureStepDescription -k PatientName -k PatientID localhost 4242
I: Requesting Association
I: Association Accepted (Max Send PDV: 16372)
I: Sending Find Request (MsgID 1)
I: Request Identifiers:
I: 
I: # Dicom-Data-Set
I: # Used TransferSyntax: Little Endian Explicit
I: (0008,0018) UI [1.2.392.200036.9125.9.0.235875605.654909696.849058840] #  54, 1 SOPInstanceUID
I: (0008,0052) CS [IMAGE]                                  #   6, 1 QueryRetrieveLevel
I: (0008,0060) CS (no value available)                     #   0, 0 Modality
I: (0010,0010) PN (no value available)                     #   0, 0 PatientName
I: (0010,0020) LO (no value available)                     #   0, 0 PatientID
I: (0020,000d) UI [1.2.840.10008.1.500817.567467863.1560288934.406684597] #  54, 1 StudyInstanceUID
I: (0020,000e) UI [1.2.392.200036.9125.3.331555024156.64663542911.19447643] #  56, 1 SeriesInstanceUID
I: (0040,0254) LO (no value available)                     #   0, 0 PerformedProcedureStepDescription
I: 
I: ---------------------------
I: Find Response: 1 (Pending)
I: 
I: # Dicom-Data-Set
I: # Used TransferSyntax: Little Endian Explicit
I: (0008,0005) CS [ISO_IR 100]                             #  10, 1 SpecificCharacterSet
I: (0008,0018) UI [1.2.392.200036.9125.9.0.235875605.654909696.849058840] #  54, 1 SOPInstanceUID
I: (0008,0052) CS [IMAGE ]                                 #   6, 1 QueryRetrieveLevel
I: (0008,0054) AE [ORTHANC ]                               #   8, 1 RetrieveAETitle
I: (0008,0060) CS [CR]                                     #   2, 1 Modality
I: (0010,0010) PN [ABCD]                                   #   4, 1 PatientName
I: (0010,0020) LO [CLU121161 ]                             #  10, 1 PatientID
I: (0020,000d) UI [1.2.840.10008.1.500817.567467863.1560288934.406684597] #  54, 1 StudyInstanceUID
I: (0020,000e) UI [1.2.392.200036.9125.3.331555024156.64663542911.19447643] #  56, 1 SeriesInstanceUID
I: (0040,0254) LO [X-RAY CHEST PA]                         #  14, 1 PerformedProcedureStepDescription
I: 
I: Received Final Find Response (Success)
I: Releasing Association
```

I'm getting the Modality, Procedure Description, PatientName using this query!

Before moving ahead, I'm adding these to my orthanc config's DicomModalities:

```
"test2" : ["MOVESCU", "172.29.0.1", 2000],
"test3" : ["GETSCU", "172.29.0.1", 2000]
```

You could alternatively only have one entry, and use that AE Title in every command using the `-aet` flag

 ### **C-Move** - [movescu](https://support.dcmtk.org/docs/movescu.html)

The way C-Move works is slightly different. It involves three parties instead of two. The first is the one issuing the C-Move command, the second is the server where the dicom file to be retrieved exists and the third is the destination AE. The issuer and destination can be the same.

To get started with C-Move, we'll have to change our docker setup a bit. Let us edit our docker file to have two orthanc instances running instead.

This is what my new `docker-compose.yml` file looks like:

```
version: "3"
services:
  pacs:
    image: jodogne/orthanc-plugins
    ports:
      - 8042:8042
      - 4242:4242
    volumes:
      - ./orthanc.json:/etc/orthanc/orthanc.json:ro
      - orthanc_db:/var/lib/orthanc/db/
    expose: 
      - 4242

  pacs2:
    image: jodogne/orthanc-plugins
    ports:
      - 8043:8043
      - 4243:4243
    volumes:
      - ./orthanc2.json:/etc/orthanc/orthanc.json:ro
      - orthanc_db2:/var/lib/orthanc/db2/
    expose:
      - 4243

volumes: 
  orthanc_db:

  orthanc_db2:
```

A different config file is used for the second Orthanc instance so that we can configure it with a different AE Title, Dicom Port and HTTP Port (4242, 8042 and 4243, 8043 respectively). [Here](/images/dicom-networking/configs/orthanc2.json) is the second config file if you want to take a look. We also have to edit the configs to enter new entries in `DicomModalities`, so that both Orthanc servers know of each other. [Here](/images/dicom-networking/configs/orthanc.json)  is a link to the first config file just in case.

With this, let's run `docker-compose up` and you should be able to access both Orthanc servers at `http://localhost:8042` and `http://localhost:8043`.

![](/images/dicom-networking/server1.png)
![](/images/dicom-networking/server2.png)

Let's try using `movescu` now. Our goal is to issue a movescu command to look for and send a dicom file from the first Orthanc server to the second Orthanc Server. 

```
(base) milind@Milinds-MacBook-Air ohif-docker-compose % movescu -v -P -k 0008,0052="PATIENT" -k PatientID="CLU121161" -aem "ORTHANC2" localhost 4242

I: Requesting Association
I: Association Accepted (Max Send PDV: 16372)
I: Sending Move Request (MsgID 1)
I: Request Identifiers:
I: 
I: # Dicom-Data-Set
I: # Used TransferSyntax: Little Endian Explicit
I: (0008,0052) CS [PATIENT]                                #   8, 1 QueryRetrieveLevel
I: (0010,0020) LO [CLU121161]                              #  10, 1 PatientID
I: 
I: Received Final Move Response (Success)
I: Releasing Association
```
It says success, let's look at our second Orthanc server again

![](/images/dicom-networking/server2after.png)

It worked! If you look at the command again, `-aem "ORTHANC2"` is used to tell the SCU to send the dicom to AE Title `ORTHANC2` from its records. If you look back at the config file, you will see that I have added an entry for the same so that the server knows all info about `ORTHANC2`.

The more challenging bit is to get your Orthanc server to send a dicom to the host machine's movescu. To get this working, add the following entry to your Orthanc server's `DicomModalities`:

`"store" : ["STORESCP", "host.docker.internal", 2001]`

This is basically letting the Orthanc server know of AE Title `STORESCP` with the address `host.docker.internal` and port `2001`. This is because we'll be running the `storescp` command that will listen on port 2001. Let's try it out now!

We'll first start with `storescp` by using the command `storescp -v 2001` (`-v` is for verbose mode). After this we will use `movescu` by using `movescu -v -P -k 0008,0052="PATIENT" -k PatientID="CLU121161" -aem "STORESCP" localhost 4242`

While the other flags might look familiar from above, `-aem "STORESCP"`  is new. It lets the Orthanc server know to send the dicom to AE Title `STORESCP` (the info of which it should already have, since we added it to the config).

**storescp**
```
(base) milind@Milinds-MacBook-Air ohif-docker-compose % storescp -v 2001
I: Association Received
I: Association Acknowledged (Max Send PDV: 16372)
I: Received Store Request (MsgID 1, CR)
RECV: .............................................................................................................................................................................................................................................................................................................................................................................................................................................................................
I: storing DICOM file: ./CR.1.2.392.200036.9125.9.0.235875605.654909696.849058840
I: Association Release
```

**movescu**
```
(base) milind@Milinds-MacBook-Air ohif-docker-compose % movescu -v -P -k 0008,0052="PATIENT" -k PatientID="CLU121161" -aem "STORESCP" localhost 4242
I: Requesting Association
I: Association Accepted (Max Send PDV: 16372)
I: Sending Move Request (MsgID 1)
I: Request Identifiers:
I: 
I: # Dicom-Data-Set
I: # Used TransferSyntax: Little Endian Explicit
I: (0008,0052) CS [PATIENT]                                #   8, 1 QueryRetrieveLevel
I: (0010,0020) LO [CLU121161]                              #  10, 1 PatientID
I: 
I: Received Final Move Response (Success)
I: Releasing Association
```





### **C-GET** - [getscu](https://support.dcmtk.org/docs/getscu.html) 
Used to retrieve DICOMs

Let's get ourselves familiar with dcmtk's `dump2dcm` [here](https://support.dcmtk.org/docs/dump2dcm.html) first. You can use it to create dcm files (to be used for querying) with certain DICOM tag values.
For Example:

	```
	(base) milind@Milinds-MacBook-Air ohif % cat test
	# contents of file test
	# request all images for the patient with ID=CLU121161

	(0008,0052) CS [PATIENT]     # QueryRetrieveLevel
	(0010,0020) LO [CLU121161]      # PatientID%      

	(base) milind@Milinds-MacBook-Air ohif % dump2dcm test query.dcm
	W: output transfer syntax unknown, assuming --write-xfer-little

	(base) milind@Milinds-MacBook-Air ohif % ls
	demo1.dcm		demo2.dcm		ohif-docker-compose	query.dcm		test
	```

Let's use the `query.dcm` file now.

Command: `getscu [options] peer port [dcmfile-in...]`

This one seems, straightforward, let's try it.

```
(base) milind@Milinds-MacBook-Air ohif % ls
demo1.dcm		demo2.dcm		ohif-docker-compose	query.dcm		test
(base) milind@Milinds-MacBook-Air ohif % getscu -v localhost 4242 query.dcm

I: Requesting Association
I: Association Accepted (Max Send PDV: 16372)
I: Sending C-GET Request (MsgID 1)
I: Received C-STORE Request (MsgID 1)
I: Sending C-STORE Response (Success)
I: Received C-GET Response (Success)
I: Final status report from last C-GET message:
I:   Number of Remaining Suboperations : 0
I:   Number of Completed Suboperations : 1
I:   Number of Failed Suboperations    : 0
I:   Number of Warning Suboperations   : 0
I: Releasing Association
(base) milind@Milinds-MacBook-Air ohif % ls
CR.1.2.392.200036.9125.9.0.235875605.654909696.849058840	demo2.dcm							query.dcm
demo1.dcm							ohif-docker-compose						test
```

The query file is being used (-k can be used too) here to look for a dicom and retrieve it.




