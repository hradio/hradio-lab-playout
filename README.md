# Docker Compose config for HRadio Playout with MPD

## First Time (build/start)

- install docker + docker-compose (tested with docker 18.09.2-ce and compose 1.23.2)
- get this repo  
- pull all available images from hub.docker.com via  `docker-compose pull`
- build odr-audioenc using `docker-compose build`.
- copy some mp3 in music/
- create all containers using `docker-compose up -d`
- open a browser to [ympd](#ympd) (defaults to port localhost:80)

## Usefull commands

`docker-compose up -d` create and start all containers  
`docker-compose down` stop and remove all containers  
`docker-compose ps` to check the status of all running containers  
`docker-compose start|stop|restart [containername]`  to interact with the containers  
`docker-compose logs [containername]` to get the logs of a container  

## Containers

### mpd

[Music Player Deamon](https://www.musicpd.org/) - The audio player for this project.  
Image used: [vimagick/mpd](https://hub.docker.com/r/vimagick/mpd)  
Config file: [./mpd/mpd.conf]

Ports |  |  
------------ | -------------  
6600 -> host:6600 | used for the [mpd protocol](https://www.musicpd.org/doc/html/protocol.html), so its also possible to use an app instead of [ympd](#ympd)
8000 -> host:8000 | used for the built-in HTTP streaming server  

The audio files to playout can be droped into the "music" folder of this repository. mpd will add them on startup to its library.

### ympd

[ympd](https://www.ympd.org/) - MPD Web GUI, can be accessed via HTML Browser over port 80.  
Image used: [knapoc/docker-ympd](https://hub.docker.com/r/knapoc/docker-ympd)

Ports |  |  
------------ | -------------  
8080 -> host:80 | used for the web gui to be acessable via browser

### ashuffle

[ashuffle](https://github.com/joshkunz/ashuffle) - an application for automatically shuffling your mpd library.  
Image used: [hradio/ashuffle](https://hub.docker.com/r/hradio/ashuffle)

Config via Env |  |  
------------ | -------------  
MPD_HOST | Hostname of mpd (defaults to mpd)  
MPD_PORT | Port for mpd (defaults to 6600)  
QUEUE_BUFFER | Specify to keep a buffer of `n` songs queued after the currently playing song. This is to support MPD features like crossfade that don't work if there are no more songs in the queue. (defaults to 10)

### edihttp

EDI-HTTP-Splitter - splits an incomming EDI stream into different streams with only one service. (source will be made available later)  
Image used: [hradio/edihttp](https://hub.docker.com/r/hradio/edihttp)

Config via Env |  |  
------------ | -------------  
UDP_INCOMING_PORT | UDP port where to expect the EDI stream (defaults to 50000)  
HTTP_OUTGOING_PORT | Port for the http server (defaults to 8187)  

Ports |  |  
------------ | -------------  
8187 -> host:8187 | http, for the split streams

### mpd-pad

MPD-PAD-Exporter - reads the current data from mpd and exports Dynamic Label Plus and the Slideshow for DAB. (source will be made available later)  
Image used: [hradio/mpd-pad](https://hub.docker.com/r/hradio/mpd-pad)

Config via Env |  |  
------------ | -------------  
HRADIO_DLPOUTPUTFILE | file path where to write the Dynamic Label Plus to
HRADIO_SLSOUTPUTPATH | folder where to put the cover.jpg for SLS
HRADIO_MEDIABASEPATH | path where to find the audio files
HRADIO_PRESENDTIME | Time in ms to create the DLP and SLS before the new song (defaults to 2000)  
HRADIO_MPDHOST | Hostname of mpd (defaults to mpd)
HRADIO_MPDPORT | Port for mpd (defaults to 6600)  

Needs to have the music folder mounted to export the images for the slideshow.
Also needs the to the ./odr-padenc/ folder mounted to write the new data to.

### odr-padenc

[ODR-PadEnc](https://github.com/Opendigitalradio/ODR-PadEnc) - is an encoder for Programme Associated Data (PAD) and includes support for MOT Slideshow and DLS/DLP  
Image used: [hradio/odr-padenc](https://hub.docker.com/r/hradio/odr-padenc)

Config via Env |  |  
------------ | -------------  
PADENC_OUTPUT | FIFO to write PAD data into
PADENC_INPUT_DLS | FIFO or file to read DLS text from
PADENC_INPUT_SLS | Directory to read images from
PADENC_DELAY | Wait DELAY seconds between each slide  
PADENC_FRAME | Frame length for uniform mode  
PADENC_PADSIZE | Set PAD size in bytes.  
PADENC_OTHERS | E.g. -R for Do not process slides integrity checks and resizing  

odr-padenc reads the files created by [mpd-pad](#mpd-pad) and adds them to a fifo inside a docker volume. This volume is also mounted in the [odr-audioenc](#odr-audioenc)

### odr-audioenc

[ODR-AudioEnc](https://github.com/Opendigitalradio/ODR-AudioEnc) - contains a DAB and DAB+ encoder  
Dockerfile used: [hradio-odr-audioenc/Dockerfile](./hradio-odr-audioenc/Dockerfile)

Config via Env |  |  
------------ | -------------  
AUDIOENC_INPUT | input url
AUDIOENC_OUTPUT | output - see mux.conf
AUDIOENC_BITRATE | Output bitrate in kbps. Must be a multiple of 8
AUDIOENC_PADSIZE | Set PAD size in bytes  
AUDIOENC_PADFILE | Set PAD data input fifo name  
AUDIOENC_OTHERS | E.g. --aaclc  

odr-audioenc reads the audio stream from [mpd](#mpd) and the pad fifo created by [odr-padenc](#odr-padenc) and sends the encoded audio to [odr-dabmux](#odr-dabmux)

### odr-dabmux

[ODR-DabMux](https://github.com/Opendigitalradio/ODR-DabMux) - is a DAB multiplexer compliant to ETSI EN 300 401.  
Image used: [hradio/odr-dabmux](https://hub.docker.com/r/hradio/odr-dabmux)  
Config file: [./odr-dabmux/mux.conf]  

odr-dabmux receives the encoded audio from [odr-audioenc](#odr-audioenc) and creates the DAB mulitplex like defined in [odr-dabmux/mux.conf] and prepares the multiplex for [odr-dabmod](#odr-dabmod). It also sends an edi stream to [edihttp](#edihttp).  

### odr-dabmod

[ODR-DabMod](https://github.com/Opendigitalradio/ODR-DabMod) - is a DAB (Digital Audio Broadcasting) modulator compliant to ETSI EN 300 401.
Image used: [hradio/odr-dabmod](https://hub.docker.com/r/hradio/odr-dabmod)  
Config file: [./odr-dabmod/mod.conf]  

odr-dabmod receives the multiplex from [odr-dabmux](#odr-dabmux) and utilizes some [hardware](#Supported-modulator-hardware) to send the DAB signal over the air.

## Supported modulator hardware

- [USRP: B100, B200, USRP2, USRP1](https://www.ettus.com)
- [LimeSDR](https://www.crowdsupply.com/lime-micro/limesdr-mini)
- [HackRF](https://greatscottgadgets.com/hackrf/)

## Other needed hardware

- 64bit linux pc. If you don't need the modulation, also Windows and docker should work.
- Antennas and cables (e.g. we made good experiences with [Antennentechnik-Bad-Blankenburg](https://www.antennensysteme.de/produkte/produkte/detailansicht/news/4929-01-stationaere-antenne-dab-biii-l-band/) )