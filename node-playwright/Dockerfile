ARG PLAYWRIGHT_VERSION=
FROM --platform=linux/amd64 mcr.microsoft.com/playwright:${PLAYWRIGHT_VERSION}jammy

ARG NODE_VERSION=20

LABEL maintainer="support@apify.com" description="Base image for Apify Actors using headless Chrome"
ENV DEBIAN_FRONTEND=noninteractive

# Copy the script for registering intermediate certificates.
COPY ./register_intermediate_certs.sh ./register_intermediate_certs.sh

# Install libs
RUN apt update \
    && apt install --fix-missing -yq --no-install-recommends procps xvfb xauth wget \
    && mkdir -p /tmp/.X11-unix && chmod 1777 /tmp/.X11-unix \
    # Uninstall system NodeJs
    && apt purge -yq nodejs \
    # Install node
    && apt update && apt install -y curl gnupg software-properties-common \
    # Add Python 3.11 repository and install it
    && add-apt-repository ppa:deadsnakes/ppa \
    && apt update \
    && apt install -y python3.11 python3.11-venv python3.11-dev python3-pip \
    && ln -s /usr/bin/python3.11 /usr/local/bin/python3 \
    && ln -s /usr/bin/python3.11 /usr/local/bin/python \
    # Add R repository for version 4.1 or newer
    && apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E298A3A825C0D65DFD57CBB651716619E084DAB9 \
    && add-apt-repository "deb https://cloud.r-project.org/bin/linux/ubuntu jammy-cran40/" \
    && apt update \
    && apt install -y r-base r-base-dev \
    libpoppler-cpp-dev \
    libcurl4-openssl-dev \
    libssl-dev \
    libxml2-dev \
    libfontconfig1-dev \
    libfreetype6-dev \
    && curl -sL https://deb.nodesource.com/setup_${NODE_VERSION}.x | bash - \
    && apt install -y nodejs \
    # The following packages are needed for the intermediate certificates to work in Firefox.
    ca-certificates \
    jq \
    wget \
    p11-kit \
    # Register cerificates
    && chmod +x ./register_intermediate_certs.sh \
    && ./register_intermediate_certs.sh \
    # Disable chrome auto updates, based on https://support.google.com/chrome/a/answer/9052345
    && mkdir -p /etc/default && echo 'repo_add_once=false' > /etc/default/google-chrome \
    # Install chrome
    && wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb -nv \
    && apt install --fix-missing -yq ./google-chrome-stable_current_amd64.deb && rm ./google-chrome-stable_current_amd64.deb \
    \
    # Add user so we don't need --no-sandbox.
    && groupadd -r myuser && useradd -r -g myuser -G audio,video myuser \
    && mkdir -p /home/myuser/Downloads \
    && chown -R myuser:myuser /home/myuser \
    \
    && mkdir -p /etc/opt/chrome/policies/managed \
    && echo '{ "CommandLineFlagSecurityWarningsEnabled": false }' > /etc/opt/chrome/policies/managed/managed_policies.json \
    && echo '{ "ComponentUpdatesEnabled": false }' > /etc/opt/chrome/policies/managed/component_update.json \
    \
    # Install R packages
    && Rscript -e 'install.packages(c("vctrs", "rlang", "magrittr", "lifecycle", "glue", "cli", "stringi", "stringr", "cpp11", "timechange", "generics", "sys", "curl", "askpass", "Rcpp", "utf8", "withr", "pkgconfig", "fansi", "R6", "pillar", "tidyselect", "tibble", "purrr", "dplyr", "openssl", "mime", "jsonlite", "rstudioapi", "tidyr", "snakecase", "lubridate", "hms", "qpdf", "xml2", "selectr", "httr", "prettyunits", "crayon", "re2", "getPass", "janitor", "listenv", "pdftools", "abjutils", "rvest", "progress"), repos="https://cran.rstudio.com/")' \
    \
    # Globally disable the update-notifier.
    && npm config --global set update-notifier false \
    \
    # Final cleanup
    # Cleanup time
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /src/*.deb \
    && apt clean -y && apt autoremove -y \
    && rm -rf /root/.npm \
    # This is needed to remove an annoying error message when running headful.
    && mkdir -p /tmp/.X11-unix && chmod 1777 /tmp/.X11-unix

# Run everything after as non-privileged user.
USER myuser
WORKDIR /home/myuser

# Point playwright to the preincluded browsers - moving browsers around increases the image size a *lot*
ENV PLAYWRIGHT_BROWSERS_PATH=/ms-playwright

# Tell the crawlee cli that we already have browers installed, so it skips installing them
ENV CRAWLEE_SKIP_BROWSER_INSTALL=1

# Copy source code and xvfb script
COPY --chown=myuser:myuser package.json main.js chrome_test.js start_xvfb_and_run_cmd.sh new_xvfb_run_cmd.sh /home/myuser/

# Sets path to Chrome executable, this is used by Apify.launchPuppeteer()
ENV APIFY_CHROME_EXECUTABLE_PATH=/usr/bin/google-chrome

# Tell Node.js this is a production environemnt
ENV NODE_ENV=production

# Enable Node.js process to use a lot of memory (Actor has limit of 32GB)
# Increases default size of headers. The original limit was 80kb, but from node 10+ they decided to lower it to 8kb.
# However they did not think about all the sites there with large headers,
# so we put back the old limit of 80kb, which seems to work just fine.
ENV NODE_OPTIONS="--max_old_space_size=30000 --max-http-header-size=80000"

# Install default dependencies, print versions of everything
RUN npm --quiet set progress=false \
    && npm install --omit=dev --omit=optional --no-package-lock --prefer-online \
    && echo "Installed NPM packages:" \
    && (npm list --omit=dev --omit=optional || true) \
    && echo "Node.js version:" \
    && node --version \
    && echo "NPM version:" \
    && npm --version \
    && echo "Google Chrome version:" \
    && bash -c "$APIFY_CHROME_EXECUTABLE_PATH --version" \
    # Overrides the dynamic library used by Firefox to determine trusted root certificates with p11-kit-trust.so, which loads the system certificates.
    && rm $PLAYWRIGHT_BROWSERS_PATH/firefox-*/firefox/libnssckbi.so \
    && ln -s /usr/lib/x86_64-linux-gnu/pkcs11/p11-kit-trust.so $(ls -d $PLAYWRIGHT_BROWSERS_PATH/firefox-*)/firefox/libnssckbi.so

# Set up xvfb

# We should you the autodisplay detection as suggested here: https://github.com/microsoft/playwright/issues/2728#issuecomment-678083619
ENV DISPLAY=:99
ENV XVFB_WHD=1920x1080x24+32
# Uncoment this line if you want to run browser in headfull mode by defautl.
# ENV APIFY_XVFB=1

# NOTEs:
# - This needs to be compatible with CLI.
# - Using CMD instead of ENTRYPOINT, to allow manual overriding
# CMD ./start_xvfb_and_run_cmd.sh && npm start --silent
CMD ./new_xvfb_run_cmd.sh npm start --silent
