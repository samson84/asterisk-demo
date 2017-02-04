# Asterisk demo
A short demo for practicing Asterisk installation, configuration, and usage inside a docker image.

This demo creates a dockerized Asterisk server, where possible to register by a SIP client. An IVR (Inetractive Voice 
Response) responses, when an extension is called, after selecting a number, a new outgoing number is dialed, and a client 
connected to it.

**Note:** This demo is created for learning purpuses, not use it in production environment. 

# About this document

The first part of this description is much more user friendly and application centric (Asterisk), the second part go into
some implementation details about the under laying container solution (Docker). 

# Asterisk

Asterisk is an open source PBX, responsible to handle telephone calls, and manage them. Typical applications are sales,
support, helpdesk. ***"Your call is really important for us, please press nine"*** type of systems. Responsible to manage 
the different telecom interfaces, initiate, handle, manipulate the calls, connect the subscribers at low level. 

Learn more here: [Asterisk]

## Usage

### Instructions for starting the server 
 1. Rename the `.env.examples` to `.env`, check its content and fill the missing credentials if necessary.
 2. Double check your 3rd party SIP providers details. Two registration is needed, one for the asterisk "trunk"
 registration (`TRUNK_*`), and an another "visting" registration to testing the outgoing call, as an endpoint 
 (`EXTERNAL_EXTENSION_NUMBER`).
 3. Execute the `docker-compose up` command --> you will see an up and running asterisk server with verbose logging.
 
### Instructions for testing

Switch off the `STUN` feature of the SIP clients for all tests and usage. The SIP clients should be in the same network 
with the Asterisk's docker's host. Zoiper VoIP client is used for testing this solution: [Zoiper]. The client must support the
G711 μ-law encoding (often referred as u-law) for the speech path. 

#### A local subscriber registers and initiate a call:
 1. Get a SIP client and log in with `6001@<asterisks_server_ip>`, where the `<asterisk_server_ip>` is the docker host machine's IP address.
 2. Use the `unsecurepassword` password to register. 
 3. Call the extension `8000` --> you should here a "Hello world" notification.
  
#### Simple A-B call:
 1. Register an another local subscriber `6002@<asterisk_server_ip>` with `unsecurepassword` to a VoIP phone.
 2. Make a test call to extension `8000`.
 3. Initiate a call from subscriber `6001` to `6002`.
 4. Anser the call, check the 2 way speech path.
 
#### Outbound call to 3rd party Voip provider
 1. Check your registration credentials to a 3rd party provider in the `.env` file.
 2. If there is some mismatch, correct it in the `.env` file, and restart the container by `docker-compose down`, `docker-compose up`
 3. Register your 3rd party "visting" account with the `EXTERNAL_EXTENSION_NUMBER` (right now only numeric username are suppoerted in the SIP uri) 
 to the 3rd party provider's network.
 4. Make a call from subscriber `6001` with the prefix `999` to the external extension number.
 5. Answer the call, check the 2 way speech path.
 
#### Using the IVR
 1. Register with your 3rd SIP provider's account to asterisk.
 2. Subscriber `6001` call the extension `8001`.
 3. Menu `2` calls the local subscriber `6002`, menu `3` calls the external provider's `EXTERNAL_EXTENSION_NUMBER`, menu `4` invokes the test 
 message. (menu `1` calls the local user `6001`).
 
## Asterisk configuration in short

### Local subscribers

| username | password         | sip uri            |   
|:---------|:-----------------|:-------------------|
| 6001     | unsecurepassword | 6001@<asterisk_ip> 
| 6002     | unsecurepassword | 6002@<asterisk_ip> 


### Extensions

| extension | purpose                               |
|-----------|---------------------------------------|
| 6001      | subscriber 6001
| 6002      | subscriber 6002
| 8000      | hello world test call
| 8001      | IVR demo menu
| 999       | external sip provider's caller prefix


# Docker

Docker is a container solution, responsible to explicit describe applications, services and its execution environment, also
isolate them from each other. Provides scaling and network management over multiple hosts. The real magic is a vivid ecosystem
created by the docker community around a bunch of pre-baked images.
   
Learn more here: [Docker]

## Paths and files
 * `./docker-build`: the build directory for the docker image.
 * `./dokcer-build/config`: The asterisk config files location. If they have a `j2` extension, will be
   processed as a jinja2 template. (--> See below.)
 * `./env.example`: lists of used environment variables to **run** the docker image. Rename to `.env`, 
 and fill with correct values to automatically process by docker-compose. (--> See below.)
 * `./docker-compose.yml`: A usage example with parameters for the image. Not used to build 
 scalable services, just an easy way to parameter the image, and set up the environment variables within.
 
## Asterisk configuration files
 * `asterisk.conf`: Basic config, set up the verbose logging and the asterisk:asterisk process owner.
 * `modules.conf`: Default module config file.
 * `extension.conf.j2`: Template file for the dial plan config, `extension.conf`. 
 * `pjpsip.conf.js2`: Template file for PJPSIP channel config, local users description, trunk config, `pjpsip.conf`.
 * `rtp.conf.j2`: defines the RTP port range, needed for Docker port binding. Templatefile for the `rtp.conf`. 
 
## Building the image
 
The docker imgage building is also includes the downloading, compiling and building from source of asterisk. The original
build script ([Respoke asterisk image]) is optimized, minimal sound files are downloaded, and the default config files are not copied
to the default directory, it should be provide by the package user.

All files from `./docker-build/config` are copied to `/etc/asterisk` during the docker image build.  

## Environment variables

Environment variables should be set up properly in the docker image, during the docker run command to work the image 
properly. These variables are listed and commented in the `.env.example` file.  

You most propably get an jinja2 or docker-compose "missing variable" or "syntax" error if something is missing or incorrect
here.

## Templates

All asterisk config (`*.conf`), and config template (`*.conf.j2`)  files are copied into the asterisk config dir (`/etc/asterisk`)
during the building of the image. 

When the docker image is actually run, the entrypoint script going through the `j2` files in
the config dir, and apply them as a jinja2 template using the `j2cli` python module. The environment variable values in the 
docker image automatically substituted to the same variable names in the template. 

For example if the `PATH` variable is exists in the docker image, and a template refers to it by
`{{PATH}}`, the template's placeholder will be replaced the environment's `PATH` value.
 
After processing a j2 file, a new file created without the j2 extension, and the original template
file is deleted from the config dir. For example the `rtp.conf.j2` become `rtp.conf`, after
the template placeholders are substituted with the proper environment variables.

More on Jinja2 templating: [Jinja2 reference]
 
## Exposed ports and netorking

 * UDP `5060` for SIP signaling: Inside the docker container asterisk is configured to use this port for SIP signaling. See `pjsip.conf.j2`.
 This port should be exposed.
 * UDP port range for RTP media stream: The port range that can be use to stream the call content should be exposed also. 
 The bottom and the top of the range is defined by environment variables (see `.env.example` file), which used in the
 `rtp.conf` config file used by asterisk's RTP channels. The binded ports should be the same on the docker container also.
 * This image is use the default `docker0` bridge for internal network.
 
## Debugging the image and the asterisk config

You can use the following docker run command after the building of the image to reach the image with an interactive 
terminal, after the entry point and the template processing has been executed.
  
`docker run -it --env-file .env -p 5060:5060/udp -p 10000:10010/udp asterisk /bin/bash`

Where `asterisk` is the image tag used for build and `.env` is the file contains the environment variabales. 

Inside the container the `asterisk -c` command starts the asterisk server, also reach the asterisk console. 
You can also start the asterisk process by `asterisk`, and after connect to it remotely using the 
`asterisk -r`.

Useful asterisk console commands:
 * Check the registration status of the local endpoints: `pjsip show aors`
 * Check the registration of the 3rd party VoIP provider: `pjsip show registrations`
 * Switch on the SIP message logging: `pjsip set logger on`
 * To check the loaded dialplan: `dialplan show`
 * To reload the dialplan: `dialplan reload`
 
Useful linux commands:
 * Check that the asterisk service is running and listening: `netstat -nap | grep asterisk`
 * Check that the ports are bound: `netstat -nap | grep 5060`
 * Check the asterisk process is running: `ps -aux | grep asterisk`
 * `ifconfig`, `cat /etc/hosts` for network setup inside the container.
 
Useful docker commands:
 * Check a running, stopped or built container details with: `docker inspect <container name>`.
 
Mount the config directory as a local directory:

`docker run -it --env-file .env -p 5060:5060/udp -p 10000:10010/udp -v $(pwd)/asterisk:/etc/asterisk asterisk /bin/bash`

Copy the config files to the `./asterisk` before executing the command, and edit them on the fly, 
without rebuild the continer. 
 
## Based on

This asterisk dockerfile is based on the wonderful respoke's work: [Respoke asterisk image]
with some custom modifications. Thanks.

# Resources
  * [Asterisk]: Open source communication framework.
  * [Docker]: Containerization framework.
  * [Respoke asterisk image]: Respoki.io implementation of the astirisk's docker containerizaion.
  * [Jinja2 reference]: templating framework written in python
  * [j2cli]: CLI interface for the Jinja2 framework.
  * [Zoiper]: Free VoIP client.

[Asterisk]: http://asterisk.org
[Docker]: http://docker.com
[Respoke asterisk image]: https://hub.docker.com/r/respoke/asterisk/
[Jinja2 reference]: http://jinja.pocoo.org/docs/2.9/templates/ 
[j2cli]: https://github.com/kolypto/j2cli
[Zoiper]: http://www.zoiper.com/

