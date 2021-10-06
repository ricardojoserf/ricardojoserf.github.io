---
layout: post
title: covert-gdrive - Control systems with Google Drive
excerpt_separator: <!--more-->
---

A program to control systems remotely by uploading files to Google Drive using Python to create the videos and the listener. It allows to upload different types of files, such as text files and videos. 

<!--more-->

### Create file to upload

It is possible to get the commands to execute from text (".txt") files or videos (".avi"). This is an example text file:

```
id > /tmp/id.txt
echo "test" > /tmp/test.txt
```

The videos can be created using *generate_video.py* entering the commands and generating the video with "exit". The video generated is called by default *output.avi* (can be updated in *config.py*): 

```
python3 generate_video.py
```

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/covert-gdrive/image2.jpg)


### Run the listener and upload a file

```
python3 main.py
```

The listener will check the Google Drive folder every 300 seconds by default (can be updated in *config.py*). First a test text is uploaded, then a test video:

![img3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/covert-gdrive/image3.jpg)

After finding there is new file, it is downloaded and the commands are executed:

![img4](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/covert-gdrive/image4.jpg)

We can see the commands output:

![img5](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/covert-gdrive/image5.jpg)

--------------------------------------------------------------------------------------

### Configuration

Update the *config.py* file:

- ***drive_folder*** (***Mandatory!!!***) - Url of open Google Drive folder.

- **temp_folder** (Optional. Default: "/tmp/"): Path where images of every frame from the video are stored, with the format *image_*X*.png*.

- **upload_seconds_delay** (Optional. Default: 300): Seconds delay until checking if a new video has been uploaded.

- **image_type** (Optional. Default: "qr_aes"): Different types of images for the video. 
	- "cleartext" creates images with the words of the commands.
	- "qr" creates QR codes with the commands.
	- "qr_aes" creates QR codes with the commands encrypted with AES.

- **aes_key** (Optional. Default: "covert-tube_2021"): Key for AES encryption, used in the "qr_aes" option.

- **generated_video_path** (Optional. Default: "output.avi"): Path of video generated with *generate_video.py*.

- **debug** (Optional. Default: True): Print messages or not.


### Installation

For all the project:

```
sudo apt install libzbar0
pip3 install Pillow opencv-python pyqrcode pypng pyzbar pycrypto
git clone https://github.com/ricardojoserf/covert-gdrive
```

### Creating a standalone binary

```
pyinstaller --onefile main.py
cp dist/main covert-gdrive
rm -rf dist build
rm main.spec
```

--------------------------------------------------------------------------------------

## Motivation

Lately I have been reading about malware using Youtube for controlling their setting remotely. For example, Casbaneiro abuses YouTube to store its C&C server domains. Each video on the channels used by the threat actor contains a description and at the end of these there is a link to a bogus Facebook or Instagram url containing the C&C server domain ([Welivesecurity blog](https://www.welivesecurity.com/2019/10/03/casbaneiro-trojan-dangerous-cooking/)). A second example is Numando, which abuses it by encrypting the data in the title of the Youtube videos ([other Welivesecurity blog](https://www.welivesecurity.com/2021/09/17/numando-latam-banking-trojan/)). 

Knowing this I decided to create a PoC to test the control of remote systems uploading videos to Youtube but, instead of using the title or the description, using the content of the video. It allows to execute any command, but it could be used to change some settings remotely. So this is just a PoC, use it for educational purposes!
