#
# Dockerfile for Fermentrack image, this version can be generated for both amd64/i386/armv7
#
ARG BUILD_FROM=hassioaddons/debian-base:3.2.3
FROM ${BUILD_FROM}
EXPOSE 8080/tcp
#
# Install the needed dependecies, some are only used at the installphase so those will be removed later.
#
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update && apt-get install --no-install-recommends -y \
  nginx \
  redis-server \
  python3.7 \
  python3-pip \
  python3-dev \
  python3-venv \
  python3-zmq \
  python3-scipy \
  python3-numpy \
  libbluetooth-dev \
  bluez \
  libcap2-bin\
  libbluetooth3 \
  ca-certificates \
  build-essential \
  git-core \
  avrdude \
  sudo \
  libffi-dev \
  libssl-dev \
  libcap2-bin \
# For debugging
  nano \
  usbutils 
#
# Install fermentrack, just make a copy of git and we will handle the run commands in the supervisor
#
WORKDIR /data/fermentrack
#
# Create the users, group and directories for git. Its needed for fermentrack to work correcly.
# User fermentrack is added to the dialout group on order to have access to firmware flashing when running as non root. 
#
RUN groupadd -g 2000 fermentrack && useradd -m -u 2001 -g fermentrack -d /data/fermentrack fermentrack
RUN groupadd -g 2002 nginx && useradd --system -M -u 2003 -g nginx nginx -s /usr/sbin/nologin
RUN usermod -a -G dialout fermentrack
USER fermentrack:fermentrack
#
# Clone the master branch of fermentrack into the image (updates will work afterwards)
#
RUN git clone --single-branch --branch dev https://github.com/thorrak/fermentrack.git /data/fermentrack/fermentrack
#
# Setup python and install the packages
#
USER root
WORKDIR /data/fermentrack
RUN pip3 install virtualenv
RUN virtualenv venv
ENV VIRTUAL_ENV="/data/fermentrack/venv"
ENV PATH="/data/fermentrack/venv/bin:/usr/bin:/usr/sbin:/bin:/sbin"
ENV PYTHONPATH=":;/data/fermentrack;/data/fermentrack/fermentrack;/data/fermentrack/venv/bin;/data/fermentrack/venv/lib/python3.8/site-packages"
ENV USE_DOCKER=true
RUN pip3 install --no-cache-dir -r /data/fermentrack/fermentrack/requirements.txt
RUN pip3 install numpy==1.18.4
#
# Setup nginx and update config to point towards fermentrack. 
#
USER root
COPY default-fermentrack /etc/nginx/sites-available/default-fermentrack
RUN rm -f /etc/nginx/sites-enabled/default
RUN ln -sf /etc/nginx/sites-available/default-fermentrack /etc/nginx/sites-enabled/default-fermentrack
RUN chown -R nginx:nginx /var/log/nginx
RUN chown -R nginx:nginx /var/lib/nginx
RUN touch /run/nginx.pid
RUN chown nginx:nginx /run/nginx.pid
#
# Setup the startup scripts and configure webservers
#
USER root
COPY entrypoint.sh /data/fermentrack/entrypoint.sh
RUN chmod 700 /data/fermentrack/entrypoint.sh
RUN rm -fR /data/fermentrack/fermentrack/data
#
# Start the supervisor
#
WORKDIR /data/fermentrack
USER root
RUN date >> /data/fermentrack/build_info
ENTRYPOINT ["/data/fermentrack/entrypoint.sh"]
CMD [""]

# Build arugments
ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_REF
ARG BUILD_VERSION

# Labels
LABEL \
    io.hass.name="Fermentrack" \
    io.hass.description="BrewPi-compatible fermentation tracking & temperature control software" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version=${BUILD_VERSION} \
    maintainer="Andy McInnes <mcinnes01@hotmail.com>" \
    org.opencontainers.image.title="Fermentrack" \
    org.opencontainers.image.description="BrewPi-compatible fermentation tracking & temperature control software" \
    org.opencontainers.image.vendor="Home Assistant Community Add-ons" \
    org.opencontainers.image.authors="Andy McInnes <mcinnes01@hotmail.com>" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.url="https://addons.community" \
    org.opencontainers.image.source="https://github.com/mcinnes01/addon-fermentrack" \
    org.opencontainers.image.documentation="https://github.com/mcinnes01/addon-fermentrack/README.md" \
    org.opencontainers.image.created=${BUILD_DATE} \
    org.opencontainers.image.revision=${BUILD_REF} \
    org.opencontainers.image.version=${BUILD_VERSION}
# EOF