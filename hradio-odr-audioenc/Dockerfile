FROM ubuntu:xenial


RUN apt-get update \
  && apt-get install -y build-essential git \
  alsa-tools alsa-utils automake libtool\
  vlc-nox libasound2-dev libjack-jackd2-dev libvlc-dev libzmq5-dev


RUN mkdir -p /root/dab/ 

WORKDIR /root/dab/
RUN git clone https://github.com/Opendigitalradio/fdk-aac.git

WORKDIR /root/dab/fdk-aac/
RUN ./bootstrap
RUN ./configure
RUN make
RUN make install

ARG git_tag=v2.3.1
WORKDIR /root/dab/
RUN git clone https://github.com/Opendigitalradio/ODR-AudioEnc.git --branch=${git_tag}
WORKDIR /root/dab/ODR-AudioEnc/
RUN ./bootstrap
RUN ./configure --enable-jack --enable-vlc --enable-alsa
RUN make
RUN make install

RUN apt-get remove -y build-essential git automake libtool && apt-get autoremove -y

RUN rm -rf /root/dab/

CMD /usr/local/bin/odr-audioenc -v $AUDIOENC_INPUT -o $AUDIOENC_OUTPUT -b $AUDIOENC_BITRATE -p $AUDIOENC_PADSIZE -P $AUDIOENC_PADFILE $AUDIOENC_OTHERS




