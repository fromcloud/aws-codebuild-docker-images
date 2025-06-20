# Copyright 2020-2024 Amazon.com, Inc. or its affiliates. All Rights Reserved.
#
# Licensed under the Amazon Software License (the "License"). You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#    http://aws.amazon.com/asl/
#
# or in the "license" file accompanying this file.
# This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied.
# See the License for the specific language governing permissions and limitations under the License.
FROM public.ecr.aws/ubuntu/ubuntu:20.04 AS core

ARG DEBIAN_FRONTEND="noninteractive"

# Install git, SSH, and other utilities
RUN set -ex \
    && echo 'Acquire::CompressionTypes::Order:: "gz";' > /etc/apt/apt.conf.d/99use-gzip-compression \
    && apt-get update \
    && apt install -y -qq apt-transport-https gnupg ca-certificates containerd curl \
    && mkdir -p /etc/containerd \
    && containerd config default > /etc/containerd/config.toml \
    && curl -LO https://github.com/containerd/nerdctl/releases/download/v1.7.6/nerdctl-1.7.6-linux-amd64.tar.gz \
    && tar -xzf nerdctl-1.7.6-linux-amd64.tar.gz \
    && mv nerdctl /usr/local/bin && rm nerdctl-1.7.6-linux-amd64.tar.gz \
    && /bin/bash -c containerd & sleep 3 && bash \
    && apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF \
    && apt-get install software-properties-common -y -qq --no-install-recommends \
    && apt-get install -y -qq --no-install-recommends openssh-client \
    && mkdir ~/.ssh \
    && mkdir -p /codebuild/image/config \
    && touch ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H github.com >> ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H bitbucket.org >> ~/.ssh/known_hosts \
    && chmod 600 ~/.ssh/known_hosts \
    && apt-get install -y -qq --no-install-recommends \
          apt-utils asciidoc autoconf automake build-essential bzip2 \
          bzr curl dirmngr docbook-xml docbook-xsl dpkg-dev \
          e2fsprogs expect fakeroot file g++ gcc gettext gettext-base \
          groff gzip iptables jq less libapr1 libaprutil1 \
          libargon2-0-dev libbz2-dev libc6-dev libcurl4-openssl-dev \
          libdb-dev libdbd-sqlite3-perl libdbi-perl libdpkg-perl \
          libedit-dev liberror-perl libevent-dev libffi-dev libgeoip-dev \
          libglib2.0-dev libhttp-date-perl libio-pty-perl libjpeg-dev \
          libkrb5-dev liblzma-dev libmagickcore-dev libmagickwand-dev \
          libmysqlclient-dev libncurses5-dev libncursesw5-dev libonig-dev \
          libpq-dev libreadline-dev libserf-1-1 libsodium-dev libsqlite3-dev libssl-dev \
          libsvn1 libsvn-perl libtcl8.6 libtidy-dev libtimedate-perl \
          libtool libwebp-dev libxml2-dev libxml2-utils libxslt1-dev \
          libyaml-dev libyaml-perl llvm locales make mlocate \
          netbase openssl patch pkg-config procps python3-configobj \
          python-openssl rsync sgml-base sgml-data \
          tar tcl tcl8.6 tk tk-dev unzip wget xfsprogs xml-core xmlto xsltproc \
          libzip5 libzip-dev vim xvfb xz-utils zip zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

ENV LC_CTYPE="C.UTF-8"

RUN useradd codebuild-user

#=======================End of layer: core  =================


FROM core AS tools

# Install Git
RUN set -ex \
   && GIT_VERSION=2.43.2 \
   && GIT_TAR_FILE=git-$GIT_VERSION.tar.gz \
   && GIT_SRC=https://github.com/git/git/archive/v${GIT_VERSION}.tar.gz  \
   && curl -L -o $GIT_TAR_FILE $GIT_SRC \
   && tar zxf $GIT_TAR_FILE \
   && cd git-$GIT_VERSION \
   && make -j4 \
   && make install prefix=/usr \
   && cd .. && rm -rf git-$GIT_VERSION \
   && rm -rf $GIT_TAR_FILE /tmp/*

# Install Firefox
RUN set -ex \
    && apt-add-repository -y "deb http://archive.canonical.com/ubuntu $(lsb_release -sc) partner" \
    && apt-get install -y -qq firefox \
    && firefox --version

# Install GeckoDriver
RUN set -ex \
    && apt-get install -y -qq firefox-geckodriver \
    && geckodriver --version

# Install Chrome
ENV CHROME_VERSION="126.0.6478.55-1"

RUN set -ex \
    && wget --no-verbose -O /tmp/chrome.deb https://dl.google.com/linux/chrome/deb/pool/main/g/google-chrome-stable/google-chrome-stable_${CHROME_VERSION}_amd64.deb \
    && (dpkg -i /tmp/chrome.deb || apt-get -fy install) \
    && rm /tmp/chrome.deb \
    && sed -i 's|HERE/chrome"|HERE/chrome" --disable-setuid-sandbox --no-sandbox|g' "/opt/google/chrome/google-chrome" \
    && google-chrome --version

# Install ChromeDriver
RUN set -ex \
    && CHROME_VERSION=`google-chrome --version | awk -F '[ .]' '{print $3"."$4"."$5"."$6}'` \
    && wget -qO /tmp/chromedriver_linux64.zip https://storage.googleapis.com/chrome-for-testing-public/${CHROME_VERSION}/linux64/chromedriver-linux64.zip \
    && unzip -q /tmp/chromedriver_linux64.zip -d /opt \
    && rm /tmp/chromedriver_linux64.zip \
    && mv /opt/chromedriver-linux64 /opt/chromedriver-${CHROME_VERSION} \
    && chmod 755 /opt/chromedriver-${CHROME_VERSION} \
    && ln -s /opt/chromedriver-${CHROME_VERSION}/chromedriver /usr/bin/chromedriver \
    && chromedriver --version

# Install stunnel
RUN set -ex \
   && STUNNEL_VERSION=5.69 \
   && STUNNEL_TAR=stunnel-$STUNNEL_VERSION.tar.gz \
   && STUNNEL_SHA256="1ff7d9f30884c75b98c8a0a4e1534fa79adcada2322635e6787337b4e38fdb81" \
   && curl -o $STUNNEL_TAR https://www.stunnel.org/archive/5.x/$STUNNEL_TAR && echo "$STUNNEL_SHA256 $STUNNEL_TAR" | sha256sum --check && tar xfz $STUNNEL_TAR \
   && cd stunnel-$STUNNEL_VERSION \
   && ./configure \
   && make -j4 \
   && make install \
   && openssl genrsa -out key.pem 2048 \
   && openssl req -new -x509 -key key.pem -out cert.pem -days 1095 -subj "/C=US/ST=Washington/L=Seattle/O=Amazon/OU=Codebuild/CN=codebuild.amazon.com" \
   && cat key.pem cert.pem >> /usr/local/etc/stunnel/stunnel.pem \
   && cd .. && rm -rf stunnel-${STUNNEL_VERSION}*

# AWS Tools
# https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html
RUN curl -sS -o /usr/local/bin/aws-iam-authenticator https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/aws-iam-authenticator \
    && curl -sS -o /usr/local/bin/kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl \
    && curl -sS -o /usr/local/bin/ecs-cli https://s3.amazonaws.com/amazon-ecs-cli/ecs-cli-linux-amd64-latest \
    && curl -sS -L https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz | tar xz -C /usr/local/bin \
    && chmod +x /usr/local/bin/kubectl /usr/local/bin/aws-iam-authenticator /usr/local/bin/ecs-cli /usr/local/bin/eksctl

# Configure SSM
RUN set -ex \
    && mkdir /tmp/ssm \
    && cd /tmp/ssm \
    && wget -q https://s3.amazonaws.com/amazon-ssm-us-east-1/latest/debian_amd64/amazon-ssm-agent.deb \
    && dpkg -i amazon-ssm-agent.deb

# Install AWS CLI v2
# https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html
RUN curl https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o /tmp/awscliv2.zip \
    && unzip -q /tmp/awscliv2.zip -d /opt \
    && /opt/aws/install --update -i /usr/local/aws-cli -b /usr/local/bin \
    && rm /tmp/awscliv2.zip \
    && rm -rf /opt/aws \
    && aws --version

# Install env tools for runtimes
# Dotnet
ENV PATH "/root/.dotnet/:/root/.dotnet/tools/:$PATH"
RUN set -ex  \
&& wget -qO /usr/local/bin/dotnet-install.sh https://dot.net/v1/dotnet-install.sh \
&& chmod +x /usr/local/bin/dotnet-install.sh

#nodejs
ARG SRC_DIR="/usr/src"
ARG N_SRC_DIR="$SRC_DIR/n"
RUN git clone https://github.com/tj/n $N_SRC_DIR \
     && cd $N_SRC_DIR && make install 

#ruby
ARG RBENV_SRC_DIR="/usr/local/rbenv"

ENV PATH="/root/.rbenv/shims:$RBENV_SRC_DIR/bin:$RBENV_SRC_DIR/shims:$PATH" \
    RUBY_BUILD_SRC_DIR="$RBENV_SRC_DIR/plugins/ruby-build"

RUN set -ex \
    && git clone https://github.com/rbenv/rbenv.git $RBENV_SRC_DIR \
    && mkdir -p $RBENV_SRC_DIR/plugins \
    && git clone https://github.com/rbenv/ruby-build.git $RUBY_BUILD_SRC_DIR \
    && sh $RUBY_BUILD_SRC_DIR/install.sh

#python
RUN curl -s -S -L https://raw.githubusercontent.com/pyenv/pyenv-installer/master/bin/pyenv-installer | bash
ENV PATH="/root/.pyenv/shims:/root/.pyenv/bin:$PATH"

#php
RUN curl -L https://raw.githubusercontent.com/phpenv/phpenv-installer/master/bin/phpenv-installer | bash
ENV PATH="/root/.phpenv/shims:/root/.phpenv/bin:$PATH"

#go
RUN git clone https://github.com/syndbg/goenv.git $HOME/.goenv
ENV PATH="/root/.goenv/shims:/root/.goenv/bin:/go/bin:$PATH"
ENV GOENV_DISABLE_GOPATH=1
ENV GOPATH="/go"

#=======================End of layer: tools  =================
FROM tools AS runtimes

#****************     .NET-CORE     *******************************************************

ENV DOTNET_31_SDK_VERSION="3.1.426"
ENV DOTNET_5_SDK_VERSION="5.0.408"
ENV DOTNET_ROOT="/root/.dotnet"

# Add .NET Core 3.1 Global Tools install folder to PATH
RUN  /usr/local/bin/dotnet-install.sh -v $DOTNET_31_SDK_VERSION \
     && dotnet --list-sdks \
     && rm -rf /tmp/*

# Add .NET Core 5 Global Tools install folder to PATH
RUN  /usr/local/bin/dotnet-install.sh -v $DOTNET_5_SDK_VERSION \
     && dotnet --list-sdks \
     && rm -rf /tmp/*

# Trigger the population of the local package cache
ENV NUGET_XMLDOC_MODE skip
RUN set -ex \
    && mkdir warmup \
    && cd warmup \
    && dotnet new \
    && cd .. \
    && rm -rf warmup \
    && rm -rf /tmp/NuGetScratch

# Install Powershell Core
# See instructions at https://docs.microsoft.com/en-us/powershell/scripting/setup/installing-powershell-core-on-linux
ARG POWERSHELL_VERSION=7.3.4
ARG POWERSHELL_DOWNLOAD_URL=https://github.com/PowerShell/PowerShell/releases/download/v$POWERSHELL_VERSION/powershell-$POWERSHELL_VERSION-linux-x64.tar.gz
ARG POWERSHELL_DOWNLOAD_SHA=E85D5544E13A924F8B2C4A5DC2D43ABE46E46633F89E8D138D39C0AAEACB9976

RUN set -ex \
    && curl -SL $POWERSHELL_DOWNLOAD_URL --output powershell.tar.gz \
    && echo "$POWERSHELL_DOWNLOAD_SHA powershell.tar.gz" | sha256sum -c - \
    && mkdir -p /opt/microsoft/powershell/$POWERSHELL_VERSION \
    && tar zxf powershell.tar.gz -C /opt/microsoft/powershell/$POWERSHELL_VERSION \
    && rm powershell.tar.gz \
    && ln -s /opt/microsoft/powershell/$POWERSHELL_VERSION/pwsh /usr/bin/pwsh
#****************     END .NET-CORE     *******************************************************


#****************      NODEJS     ****************************************************

ENV NODE_12_VERSION="12.22.12"
ENV NODE_14_VERSION="14.21.3"

RUN     n $NODE_14_VERSION && npm install --save-dev -g -f grunt && npm install --save-dev -g -f grunt-cli && npm install --save-dev -g -f webpack \
     && n $NODE_12_VERSION && npm install --save-dev -g -f grunt && npm install --save-dev -g -f grunt-cli && npm install --save-dev -g -f webpack \
     && curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
     && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
     && apt-get update && apt-get install -y -qq --no-install-recommends yarn \
     && yarn --version \
     && cd / && rm -rf $N_SRC_DIR && rm -rf /tmp/*

#****************      END NODEJS     ****************************************************

#**************** RUBY *********************************************************

ENV RUBY_26_VERSION="2.6.10"
ENV RUBY_27_VERSION="2.7.8"

RUN rbenv install $RUBY_26_VERSION && rm -rf /tmp/* \
    && rbenv install $RUBY_27_VERSION && rm -rf /tmp/* \
    && rbenv global $RUBY_27_VERSION && ruby -v

#**************** END RUBY *****************************************************

#**************** PYTHON *****************************************************
ENV PYTHON_37_VERSION="3.7.17"
ENV PYTHON_38_VERSION="3.8.17"
ENV PYTHON_39_VERSION="3.9.17"

ARG PYTHON_PIP_VERSION=21.3.1
ENV PYYAML_VERSION=5.4.1

COPY tools/runtime_configs/python/$PYTHON_37_VERSION /root/.pyenv/plugins/python-build/share/python-build/$PYTHON_37_VERSION
RUN   env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install $PYTHON_37_VERSION && rm -rf /tmp/*
RUN   pyenv global  $PYTHON_37_VERSION
RUN set -ex \
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
    && pip3 install --no-cache-dir --upgrade "PyYAML==$PYYAML_VERSION" \
    && pip3 install --no-cache-dir --upgrade 'setuptools==57.5.0' wheel aws-sam-cli boto3 pipenv virtualenv --use-feature=2020-resolver

COPY tools/runtime_configs/python/$PYTHON_38_VERSION /root/.pyenv/plugins/python-build/share/python-build/$PYTHON_38_VERSION
RUN   env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install $PYTHON_38_VERSION && rm -rf /tmp/*
RUN   pyenv global  $PYTHON_38_VERSION
RUN set -ex \
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
    && pip3 install --no-cache-dir --upgrade "PyYAML==$PYYAML_VERSION" \
    && pip3 install --no-cache-dir --upgrade 'setuptools==57.5.0' wheel aws-sam-cli boto3 pipenv virtualenv --use-feature=2020-resolver

COPY tools/runtime_configs/python/$PYTHON_39_VERSION /root/.pyenv/plugins/python-build/share/python-build/$PYTHON_39_VERSION
RUN   env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install $PYTHON_39_VERSION && rm -rf /tmp/*
RUN   pyenv global  $PYTHON_39_VERSION
RUN set -ex \
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
    && pip3 install --no-cache-dir --upgrade "PyYAML==$PYYAML_VERSION" \
    && pip3 install --no-cache-dir --upgrade 'setuptools==57.5.0' wheel aws-sam-cli boto3 pipenv virtualenv --use-feature=2020-resolver

#**************** END PYTHON *****************************************************

#****************      PHP     ****************************************************
ENV PHP_73_VERSION="7.3.33"
ENV PHP_74_VERSION="7.4.33"
ENV PHP_80_VERSION="8.0.29"

RUN curl -L https://raw.githubusercontent.com/phpenv/phpenv-installer/master/bin/phpenv-installer | bash
ENV PATH="/root/.phpenv/shims:/root/.phpenv/bin:$PATH"
COPY tools/runtime_configs/php/$PHP_73_VERSION /root/.phpenv/plugins/php-build/share/php-build/definitions/$PHP_73_VERSION
RUN phpenv install $PHP_73_VERSION && rm -rf /tmp/* && phpenv global $PHP_73_VERSION
RUN echo "memory_limit = 1G;" >> "/root/.phpenv/versions/$PHP_73_VERSION/etc/conf.d/memory.ini"

COPY tools/runtime_configs/php/$PHP_74_VERSION /root/.phpenv/plugins/php-build/share/php-build/definitions/$PHP_74_VERSION
RUN phpenv install $PHP_74_VERSION && rm -rf /tmp/* && phpenv global $PHP_74_VERSION
RUN echo "memory_limit = 1G;" >> "/root/.phpenv/versions/$PHP_74_VERSION/etc/conf.d/memory.ini"

COPY tools/runtime_configs/php/$PHP_80_VERSION /root/.phpenv/plugins/php-build/share/php-build/definitions/$PHP_80_VERSION
RUN phpenv install $PHP_80_VERSION && rm -rf /tmp/* && phpenv global $PHP_80_VERSION
RUN echo "memory_limit = 1G;" >> "/root/.phpenv/versions/$PHP_80_VERSION/etc/conf.d/memory.ini"

# Install Composer globally
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin --filename=composer
#****************      END PHP     ****************************************************

#****************     GOLANG     ****************************************************
ENV GOLANG_15_VERSION="1.15.15"
ENV GOLANG_16_VERSION="1.16.15"

RUN goenv install $GOLANG_15_VERSION && rm -rf /tmp/* && \
    goenv install $GOLANG_16_VERSION && rm -rf /tmp/* && \
    goenv global  $GOLANG_15_VERSION

RUN go get -u github.com/golang/dep/cmd/dep
#****************      END GOLANG     *******************************

#=======================End of layer: runtimes  =================

FROM runtimes AS runtimes_n_corretto

#****************      JAVA     ****************************************************
ENV JAVA_11_HOME="/usr/lib/jvm/java-11-amazon-corretto"
ENV JDK_11_HOME="/usr/lib/jvm/java-11-amazon-corretto"
ENV JRE_11_HOME="/usr/lib/jvm/java-11-amazon-corretto"
ENV JAVA_8_HOME="/usr/lib/jvm/java-1.8.0-amazon-corretto"
ENV JDK_8_HOME="/usr/lib/jvm/java-1.8.0-amazon-corretto"
ENV JRE_8_HOME="/usr/lib/jvm/java-1.8.0-amazon-corretto/jre"
ARG ANT_VERSION=1.10.13
ARG MAVEN_HOME="/opt/maven"
ARG MAVEN_VERSION=3.6.3
ARG INSTALLED_GRADLE_VERSIONS="5.6.4 6.9.4"
ARG GRADLE_VERSION=5.6.4
ARG SBT_VERSION=1.6.2
ARG GRADLE_PATH="$SRC_DIR/gradle"
ARG ANT_DOWNLOAD_SHA512="de4ac604629e39a86a306f0541adb3775596909ad92feb8b7de759b1b286417db24f557228737c8b902d6abf722d2ce5bb0c3baa3640cbeec3481e15ab1958c9"
ARG MAVEN_DOWNLOAD_SHA512="c35a1803a6e70a126e80b2b3ae33eed961f83ed74d18fcd16909b2d44d7dada3203f1ffe726c17ef8dcca2dcaa9fca676987befeadc9b9f759967a8cb77181c0"
ARG GRADLE_DOWNLOADS_SHA256="abc10bcedb58806e8654210f96031db541bcd2d6fc3161e81cb0572d6a15e821 5.6.4\n84b50e7b380e9dc9bbc81e30a8eb45371527010cf670199596c86875f774b8b0 6.9.4"
ARG SBT_DOWNLOAD_SHA256="637637b6c4e6fa04ab62cd364061e32b12480b09001cd23303df62b36fadd440"
ENV LOG4J_UNSAFE_VERSIONS="2.11.1 1.2.8"

ARG MAVEN_CONFIG_HOME="/root/.m2"

ENV JAVA_HOME="$JAVA_11_HOME" \
    JDK_HOME="$JDK_11_HOME" \
    JRE_HOME="$JRE_11_HOME"

ENV PATH="${PATH}:/opt/tools"

RUN set -ex \
    && apt-get update \
    && apt-get install -y -qq software-properties-common apt-utils sudo \
    # Install Corretto 8
    && wget -O - https://apt.corretto.aws/corretto.key | sudo gpg --dearmor -o /usr/share/keyrings/corretto-keyring.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/corretto-keyring.gpg] https://apt.corretto.aws stable main" | sudo tee /etc/apt/sources.list.d/corretto.list \
    && apt-get update \
    && apt-get install -y -qq java-1.8.0-amazon-corretto-jdk \
    && apt-get install -y -qq --no-install-recommends ca-certificates-java \
    # Ensure Java cacerts symlink points to valid location
    && update-ca-certificates -f \
    && dpkg --add-architecture i386 \
    && apt-get update \
    && apt-get install -y -qq --force-yes libc6-i386 \
       lib32stdc++6 lib32gcc1 lib32ncurses6 \
       lib32z1 libqt5widgets5

RUN set -ex \
    # Install Corretto 11
    # Note: We will use update-alternatives to make sure JDK11 has higher priority for all the tools
    && apt-get install -y -qq java-11-amazon-corretto-jdk \
    && for tool_path in $JAVA_HOME/bin/*; do \
          tool=`basename $tool_path`; \
          update-alternatives --install /usr/bin/$tool $tool $tool_path 10000; \
          update-alternatives --set $tool $tool_path; \
        done \
     && rm $JAVA_HOME/lib/security/cacerts && ln -s /etc/ssl/certs/java/cacerts $JAVA_HOME/lib/security/cacerts \
    # Install Ant
    && curl -LSso /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz https://archive.apache.org/dist/ant/binaries/apache-ant-$ANT_VERSION-bin.tar.gz  \
    && echo "$ANT_DOWNLOAD_SHA512 /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz" | sha512sum -c - \
    && tar -xzf /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz -C /opt \
    && rm /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz \
    && update-alternatives --install /usr/bin/ant ant /opt/apache-ant-$ANT_VERSION/bin/ant 10000

RUN set -ex \
    # Install Maven
    && mkdir -p $MAVEN_HOME \
    && curl -LSso /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz https://archive.apache.org/dist/maven/maven-3/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz \
    && echo "$MAVEN_DOWNLOAD_SHA512 /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz" | sha512sum -c - \
    && tar xzf /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz -C $MAVEN_HOME --strip-components=1 \
    && rm /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz \
    && update-alternatives --install /usr/bin/mvn mvn /opt/maven/bin/mvn 10000 \
    && mkdir -p $MAVEN_CONFIG_HOME \
    # Install Gradle
    && mkdir -p $GRADLE_PATH \
    && for version in $INSTALLED_GRADLE_VERSIONS; do { \
       wget -q "https://services.gradle.org/distributions/gradle-$version-all.zip" -O "$GRADLE_PATH/gradle-$version-all.zip" \
       && unzip -q "$GRADLE_PATH/gradle-$version-all.zip" -d /usr/local \
       && echo "$GRADLE_DOWNLOADS_SHA256" | grep "$version" | sed "s|$version|$GRADLE_PATH/gradle-$version-all.zip|" | sha256sum -c - \
       && rm "$GRADLE_PATH/gradle-$version-all.zip" \
       && mkdir "/tmp/gradle-$version" \
       && "/usr/local/gradle-$version/bin/gradle" -p "/tmp/gradle-$version" wrapper \
       # Android Studio uses the "-all" distribution for it's wrapper script.
       && perl -pi -e "s/gradle-$version-bin.zip/gradle-$version-all.zip/" "/tmp/gradle-$version/gradle/wrapper/gradle-wrapper.properties" \
       && "/tmp/gradle-$version/gradlew" -p "/tmp/gradle-$version" init \
       && rm -rf "/tmp/gradle-$version" \
       && if [ "$version" != "$GRADLE_VERSION" ]; then rm -rf "/usr/local/gradle-$version"; fi; \
     }; done \
    # Install default GRADLE_VERSION to path
    && ln -s /usr/local/gradle-$GRADLE_VERSION/bin/gradle /usr/bin/gradle \
    && rm -rf $GRADLE_PATH \
    # Install SBT
    && curl -fSL "https://github.com/sbt/sbt/releases/download/v${SBT_VERSION}/sbt-${SBT_VERSION}.tgz" -o sbt.tgz \
    && echo "${SBT_DOWNLOAD_SHA256} *sbt.tgz" | sha256sum -c - \
    && tar xzf sbt.tgz -C /usr/local/bin/ \
    && rm sbt.tgz \
    && for version in $LOG4J_UNSAFE_VERSIONS; do find / -name log4j*-$version.jar | xargs rm -f; done

ENV PATH "/usr/local/bin/sbt/bin:$PATH"
RUN sbt version -Dsbt.rootdir=true
# Cleanup
RUN rm -fr /var/lib/apt/lists/* /tmp/* /var/tmp/* \
    && apt-get clean
#****************     END JAVA     ****************************************************

#****************        DOCKER    *********************************************
ARG DOCKER_BUCKET="download.docker.com"
ARG DOCKER_CHANNEL="stable"
ARG DIND_COMMIT="3b5fac462d21ca164b3778647420016315289034"
ARG DOCKER_COMPOSE_VERSION="1.29.2"
ARG SRC_DIR="/usr/src"

ARG DOCKER_SHA256="AB91092320A87691A1EAF0225B48585DB9C69CFF0ED4B0F569F744FF765515E3"
ARG DOCKER_VERSION="20.10.24"

# Install Docker
RUN set -ex \
    && curl -fSL "https://${DOCKER_BUCKET}/linux/static/${DOCKER_CHANNEL}/x86_64/docker-${DOCKER_VERSION}.tgz" -o docker.tgz \
    && echo "${DOCKER_SHA256} *docker.tgz" | sha256sum -c - \
    && tar --extract --file docker.tgz --strip-components 1  --directory /usr/local/bin/ \
    && rm docker.tgz \
    && docker -v \
    # replace runc package to resolve CVE-2024-21626
    && apt-get update && apt-get -y install runc \
    && rm -f /usr/local/bin/runc \
    && ln -s /usr/sbin/runc /usr/local/bin/runc \
    && runc -v \
    # set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
    && addgroup dockremap \
    && useradd -g dockremap dockremap \
    && echo 'dockremap:165536:65536' >> /etc/subuid \
    && echo 'dockremap:165536:65536' >> /etc/subgid \
    && wget -q "https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind" -O /usr/local/bin/dind \
    && curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-Linux-x86_64 > /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/dind /usr/local/bin/docker-compose \
    # Ensure docker-compose works
    && docker-compose version

VOLUME /var/lib/docker
#*********************** END  DOCKER  ****************************

#=======================End of layer: corretto  =================
FROM runtimes_n_corretto AS std_v5

# Activate runtime versions specific to image version.
RUN n $NODE_14_VERSION
RUN pyenv  global $PYTHON_39_VERSION
RUN phpenv global $PHP_80_VERSION
RUN rbenv  global $RUBY_27_VERSION
RUN goenv global  $GOLANG_15_VERSION

# Configure SSH
COPY ssh_config /root/.ssh/config
COPY runtimes.yml /codebuild/image/config/runtimes.yml
COPY dockerd-entrypoint.sh /usr/local/bin/dockerd-entrypoint.sh
COPY legal/bill_of_material.txt /usr/share/doc/bill_of_material.txt
COPY amazon-ssm-agent.json /etc/amazon/ssm/amazon-ssm-agent.json

ENTRYPOINT ["/usr/local/bin/dockerd-entrypoint.sh"]

#=======================END of STD:5.0  =================
