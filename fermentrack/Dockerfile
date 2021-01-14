ARG BUILD_FROM=hassioaddons/debian-base:3.2.3
# hadolint ignore=DL3006
FROM ${BUILD_FROM}
EXPOSE 80/tcp

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Setup base system
ARG BUILD_ARCH=amd64
RUN \
    ARCH="${BUILD_ARCH}" \
    && if [ "${BUILD_ARCH}" = "aarch64" ]; then ARCH="arm64"; fi \
    && if [ "${BUILD_ARCH}" = "armv7" ]; then ARCH="armhf"; fi \
    \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        libfontconfig1=2.13.1-2 \
        memcached=1.5.6-1.1 \
        nginx=1.14.2-2+deb10u3 \
        redis-server \
        python3 \
        python3-pip \
        python3-dev \
        python3-venv \
        python3-zmq \
        python3-scipy \
        python3-numpy \
        supervisor \
        libbluetooth-dev \
        bluez \
        libcap2-bin\
        libbluetooth3 \
        ca-certificates \
        build-essential \
        git-core \
        avrdude \
        nano \
    \
    && if [ "${BUILD_ARCH}" = "amd64" ]; then \
        apt-get install -y --no-install-recommends \
            libasound2=1.1.8-1 \
            libcups2=2.2.10-6+deb10u3  \
            libdbus-1-3=1.12.20-0+deb10u1 \
            libglib2.0-0=2.58.3-2+deb10u2 \
            libgtk-3-0=3.24.5-1 \
            libnss3=2:3.42.1-1+deb10u3 \
            libx11-6=2:1.6.7-1+deb10u1 \
            libx11-xcb1=2:1.6.7-1+deb10u1 \
            libxcomposite1=1:0.4.4-2 \
            libxcursor1=1:1.1.15-2 \
            libxdamage1=1:1.1.4-3+b3 \
            libxext6=2:1.3.3-1+b2 \
            libxfixes3=1:5.0.3-1 \
            libxi6=2:1.7.9-1 \
            libxrandr2=2:1.5.1-1 \
            libxrender1=1:0.9.10-1 \
            libxss1=1:1.2.3-1 \
            libxtst6=2:1.2.3-1 \
    fi \
    \
    && rm -fr \
        /tmp/* \
        /var/{cache,log}/* \
        /var/lib/apt/lists/* \
        /etc/nginx \
    \
    && mkdir -p /var/log/nginx \
    && touch /var/log/nginx/error.log

#
# Create the fermentrack user, group and directories for git. Its needed for fermentrack to work correcly
#
RUN groupadd -g 2000 fermentrack && useradd -m -u 2001 -g fermentrack fermentrack
#
# Install fermentrack, just make a copy of git and we will handle the run commands in the supervisor
#
WORKDIR /home/fermentrack
USER fermentrack:fermentrack
#
# Use this for using the master branch from thorrak (beware that running git update from the repository will break the installation)
#RUN git clone https://github.com/thorrak/fermentrack.git /home/fermentrack/fermentrack
#
# Use my development branch to include changes that blocks git upgrades that otherwise will break the setup (especially db location). 
# I will keep this branch up to date with the thorrak master branch until I can test/merge the needed changes.
RUN git clone -b docker https://github.com/mp-se/fermentrack.git /home/fermentrack/fermentrack
#
RUN /home/fermentrack/fermentrack/utils/make_secretsettings.sh
#
# Move the database to a new location to make it easier to mount volumes. docker does not behave well when working with single files.
#
RUN mv /home/fermentrack/fermentrack/fermentrack_django/settings.py /home/fermentrack/fermentrack/fermentrack_django/settings.py2
RUN sed "s/db.sqlite3/db\/db.sqlite3/" /home/fermentrack/fermentrack/fermentrack_django/settings.py2 > /home/fermentrack/fermentrack/fermentrack_django/settings.py
RUN rm /home/fermentrack/fermentrack/fermentrack_django/settings.py2
#
RUN mv /home/fermentrack/fermentrack/app/management/commands/fix_sqlite_for_django_2.py /home/fermentrack/fermentrack/app/management/commands/fix_sqlite_for_django_2.py2
RUN sed "s/db.sqlite3/db\/db.sqlite3/" /home/fermentrack/fermentrack/app/management/commands/fix_sqlite_for_django_2.py2 > /home/fermentrack/fermentrack/app/management/commands/fix_sqlite_for_django_2.py
RUN rm /home/fermentrack/fermentrack/app/management/commands/fix_sqlite_for_django_2.py2
#
RUN mkdir /home/fermentrack/fermentrack/db
#
# Install fermentrack tools, we need the config files for nginx
#
WORKDIR /home/fermentrack
USER fermentrack:fermentrack
RUN git clone https://github.com/thorrak/fermentrack-tools.git /home/fermentrack/fermentrack-tools
USER root
RUN sed "s/brewpiuser/fermentrack/" /home/fermentrack/fermentrack-tools/nginx-configs/default-fermentrack > /etc/nginx/sites-available/default-fermentrack
RUN rm -f /etc/nginx/sites-enabled/default
RUN ln -sf /etc/nginx/sites-available/default-fermentrack /etc/nginx/sites-enabled/default-fermentrack
#
# Setup python and install the packages
#
USER root
WORKDIR /home/fermentrack
RUN pip install virtualenv
RUN virtualenv venv
ENV VIRTUAL_ENV="/home/fermentrack/venv"
ENV PATH="/home/fermentrack/venv/bin:/usr/bin"
ENV HOME="/home/fermentrack"
ENV SHELL="/bin/bash"
ENV PYTHONPATH=":/home/fermentrack/fermentrack;/home/fermentrack/venv/bin;/home/fermentrack/venv/lib/python3.8/site-packages"
ENV USER=fermentrack
ENV DOCKER=1
RUN pip install --no-cache-dir -r /home/fermentrack/fermentrack/requirements.txt
USER fermentrack
WORKDIR /home/fermentrack/fermentrack
RUN python manage.py collectstatic --noinput 
#
# Setup the supervisor to manage all the processes
#
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
#
# Export the mount points
#
VOLUME ["/home/fermentrack/fermentrack/log", "/home/fermentrack/fermentrack/data", "/home/fermentrack/fermentrack/db"]
#
# Cleanup, we remove git since the docker image should not be updated this way.
#
USER root
RUN rm -fr /home/fermentrack/fermentrack-tools
RUN apt-get remove -y \
  libbluetooth-dev \
  ca-certificates \
  build-essential
#RUN apt-get cleanup -y
#
# Start the supervisor
#
WORKDIR /home/fermentrack/fermentrack
USER root
CMD ["/usr/bin/supervisord"]
# EOF

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
