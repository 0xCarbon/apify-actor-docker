ARG NODE_VERSION=20
# Use bookworm to be consistent across node versions.
FROM node:${NODE_VERSION}-bookworm-slim

LABEL maintainer="support@apify.com" description="Base image for Apify Actors using Camoufox"
ENV DEBIAN_FRONTEND=noninteractive

COPY ./register_intermediate_certs.sh ./register_intermediate_certs.sh

# Install system dependencies, Python, R, and browser deps
RUN sh -c 'echo "deb http://ftp.us.debian.org/debian bookworm main non-free" >> /etc/apt/sources.list.d/fonts.list' \
    && apt update \
    && apt install -y --no-install-recommends \
        procps ca-certificates jq wget curl gnupg p11-kit xauth xvfb \
        python3 python3-venv python3-dev python3-pip \
    && chmod +x ./register_intermediate_certs.sh \
    && ./register_intermediate_certs.sh \
    && gpg --keyserver keyserver.ubuntu.com --recv-key '95C0FAF38DB3CCAD0C080A7BDC78B2DDEABC47B7' \
    && gpg --export '95C0FAF38DB3CCAD0C080A7BDC78B2DDEABC47B7' > /usr/share/keyrings/r-project.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/r-project.gpg] https://cloud.r-project.org/bin/linux/debian bookworm-cran40/" > /etc/apt/sources.list.d/r-project.list \
    && apt update \
    && apt install -y r-base r-base-dev \
        libpoppler-cpp-dev libcurl4-openssl-dev libssl-dev \
        libxml2-dev libfontconfig1-dev libfreetype6-dev \
    && groupadd -r myuser && useradd -r -g myuser -G audio,video myuser \
    && mkdir -p /home/myuser/Downloads \
    && chown -R myuser:myuser /home/myuser \
    && npm config --global set update-notifier false \
    && npm install -g pnpm@8.15.9 \
    && PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 npm_config_ignore_scripts=1 npx playwright install-deps firefox chromium \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /src/*.deb \
    && apt clean -y && apt autoremove -y \
    && rm -rf /root/.npm \
    && mkdir -p /tmp/.X11-unix && chmod 1777 /tmp/.X11-unix

# # Install R packages
# RUN Rscript -e 'install.packages(c("vctrs", "rlang", "magrittr", "lifecycle", "glue", "cli", "stringi", "stringr", "cpp11", "timechange", "generics", "sys", "curl", "askpass", "Rcpp", "utf8", "withr", "pkgconfig", "fansi", "R6", "pillar", "tidyselect", "tibble", "purrr", "dplyr", "openssl", "mime", "jsonlite", "rstudioapi", "tidyr", "snakecase", "lubridate", "hms", "qpdf", "xml2", "selectr", "httr", "prettyunits", "crayon", "re2", "getPass", "janitor", "listenv", "pdftools", "abjutils", "rvest", "progress"), repos="https://cran.rstudio.com/")'

# Run everything after as non-privileged user.
USER myuser
WORKDIR /home/myuser

ENV PLAYWRIGHT_BROWSERS_PATH=/home/myuser/pw-browsers

# Tell the crawlee cli that we already have browers installed, so it skips installing them
ENV CRAWLEE_SKIP_BROWSER_INSTALL=1

# Copy source code and xvfb script
COPY --chown=myuser:myuser package.json main.js check-playwright-version.mjs firefox_test.js start_xvfb_and_run_cmd.sh new_xvfb_run_cmd.sh xvfb-entrypoint.sh /home/myuser/

# Tell Node.js this is a production environemnt
ENV NODE_ENV=production

# Enable Node.js process to use a lot of memory (Actor has limit of 32GB)
# Increases default size of headers. The original limit was 80kb, but from node 10+ they decided to lower it to 8kb.
# However they did not think about all the sites there with large headers,
# so we put back the old limit of 80kb, which seems to work just fine.
ENV NODE_OPTIONS="--max_old_space_size=30000 --max-http-header-size=80000"

# Install default dependencies, print versions of everything
RUN npm --quiet set progress=false \
    && PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 npm install --omit=dev --omit=optional --no-package-lock --prefer-online \
    && PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 npm install --no-package-lock --prefer-online --no-save impit@latest \
    && echo "Installed NPM packages:" \
    && (npm list --omit=dev --omit=optional || true) \
    && echo "Node.js version:" \
    && node --version \
    && echo "NPM version:" \
    && npm --version \
    && npx camoufox-js fetch \
    && npx playwright@1.46.1 install chromium \
    && rm -f /home/myuser/.cache/camoufox/libnssckbi.so \
    && ln -s "$(find /usr/lib -name p11-kit-trust.so -print -quit)" /home/myuser/.cache/camoufox/libnssckbi.so

ENV APIFY_DEFAULT_BROWSER_PATH=/home/myuser/.cache/camoufox/camoufox-bin

# Prevent installing of browsers by future `npm install`.
ENV PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1

# Set up xvfb
ENV XVFB_WHD=1920x1080x24+32

# The entrypoint script will be the one handling the CMD passed in, and will always wrap it into xvfb-run
ENTRYPOINT ["/home/myuser/xvfb-entrypoint.sh"]

# NOTEs:
# - This needs to be compatible with CLI.
# - Using CMD instead of ENTRYPOINT, to allow manual overriding
CMD ["npm", "start", "--silent"]
