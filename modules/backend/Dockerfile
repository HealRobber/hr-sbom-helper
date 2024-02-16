FROM alpine:3.19 AS base

# Install required packages
RUN apk update
# Install required packages
RUN apk add --no-cache ca-certificates less ncurses-terminfo-base krb5-libs libgcc libintl libstdc++ \
    tzdata userspace-rcu zlib icu-libs unzip wget curl

# ============================================================================ #
# Install PowerShell
# ============================================================================ #
# Download powershell
RUN curl -L https://github.com/PowerShell/PowerShell/releases/download/v7.3.3/powershell-7.3.3-linux-alpine-x64.tar.gz -o /tmp/powershell.tar.gz
# Create the target folder where powershell will be placed
RUN mkdir -p /opt/microsoft/powershell/7
# Expand powershell to the target folder
RUN tar zxf /tmp/powershell.tar.gz -C /opt/microsoft/powershell/7
# Create the symbolic link that points to pwsh
RUN ln -s /opt/microsoft/powershell/7/pwsh /usr/bin/pwsh

ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8
ENV TZ=Etc/UTC
ARG ZULU_KEY_SHA256=6c6393d4755818a15cf055a5216cffa599f038cd508433faed2226925956509a

RUN wget --quiet https://cdn.azul.com/public_keys/alpine-signing@azul.com-5d5dc44c.rsa.pub -P /etc/apk/keys/ && \
    echo "${ZULU_KEY_SHA256}  /etc/apk/keys/alpine-signing@azul.com-5d5dc44c.rsa.pub" | sha256sum -c - && \
    apk --repository https://repos.azul.com/zulu/alpine --no-cache add zulu17-jdk~=17.0.8 tzdata

ENV JAVA_HOME=/usr/lib/jvm/zulu17
ENV PATH="$PATH:$JAVA_HOME/bin"

FROM base AS gradle
CMD ["gradle"]

ENV GRADLE_HOME /opt/gradle

RUN set -o errexit -o nounset \
    && echo "Adding gradle user and group" \
    && addgroup --system --gid 1000 gradle \
    && adduser --system --ingroup gradle --uid 1000 --shell /bin/ash gradle \
    && mkdir /home/gradle/.gradle \
    && chown -R gradle:gradle /home/gradle \
    && echo "Symlinking root Gradle cache to gradle Gradle cache" \
    && ln -s /home/gradle/.gradle /root/.gradle

VOLUME /home/gradle/.gradle

WORKDIR /home/gradle

RUN set -o errexit -o nounset \
    && echo "Installing VCSes" \
    && apk add --no-cache \
      git \
      git-lfs \
      mercurial \
      subversion \
    \
    && echo "Testing VCSes" \
    && which git \
    && which git-lfs \
    && which hg \
    && which svn

ENV GRADLE_VERSION 8.1.1
ARG GRADLE_DOWNLOAD_SHA256=e111cb9948407e26351227dabce49822fb88c37ee72f1d1582a69c68af2e702f
RUN set -o errexit -o nounset \
    && echo "Downloading Gradle" \
    && wget --no-verbose --output-document=gradle.zip "https://services.gradle.org/distributions/gradle-${GRADLE_VERSION}-bin.zip" \
    \
    && echo "Checking download hash" \
    && echo "${GRADLE_DOWNLOAD_SHA256} *gradle.zip" | sha256sum -c - \
    \
    && echo "Installing Gradle" \
    && unzip gradle.zip \
    && rm gradle.zip \
    && mv "gradle-${GRADLE_VERSION}" "${GRADLE_HOME}/" \
    && ln -s "${GRADLE_HOME}/bin/gradle" /usr/bin/gradle \
    \
    && echo "Testing Gradle installation" \
    && gradle --version

FROM gradle AS app
WORKDIR /app
COPY . .
RUN gradle clean build --no-daemon --info

FROM app AS sbom-helper

# Download sbom-utility
RUN curl -L https://github.com/CycloneDX/sbom-utility/releases/download/v0.15.0/sbom-utility-v0.15.0-linux-amd64.tar.gz -o /tmp/sbom-utility.tar.gz
RUN mkdir -p /utility/sbom-utility
RUN tar zxf /tmp/sbom-utility.tar.gz -C /utility/sbom-utility
RUN export PATH="$PATH:/utility/sbom-utility"; echo $PATH
# Download & Install Grype
RUN mkdir -p /utility/grype
RUN curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /utility/grype
RUN export PATH="$PATH:/utility/grype"; echo $PATH

WORKDIR /app
COPY --from=app ./app/build/libs/backend.jar .

CMD ["java", "-jar", "backend.jar"]
