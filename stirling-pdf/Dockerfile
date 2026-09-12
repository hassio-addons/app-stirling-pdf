ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:9.4.0
# hadolint ignore=DL3006
FROM ${BUILD_FROM}

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ENV \
    JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=70 -XX:+ExitOnOutOfMemoryError -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+UseStringDeduplication -Djava.awt.headless=true" \
    PATH="/opt/venv/bin:${PATH}" \
    VIRTUAL_ENV=/opt/venv

# Stirling PDF is a Spring Boot application that shells out to a row of command
# line tools for the work Java cannot do itself, and refuses the matching tool
# in the interface when one of them is absent. Which package answers for which
# tool group:
#
#   libreoffice-*-nogui + python3-uno  office documents, through unoserver
#   ghostscript, qpdf, poppler-utils   the PDF operations proper
#   tesseract-ocr, ocrmypdf, unpaper   OCR, and the page cleanup it does first
#   imagemagick                        the image path through Compress PDF
#   fontforge-nox                      Type 3 font conversion, CFF to TrueType
#   fonts-*                            what LibreOffice draws a document with
#
# Two are deliberately absent. Calibre answers for one endpoint, PDF to EPUB,
# and costs 833MB installed because it brings Qt6 WebEngine and PyQt6 with it.
# rar answers for PDF to CBR and is non-free, so it cannot ship in an image
# that gets redistributed. Stirling PDF probes for both at startup and greys
# out what it cannot find, so their absence is reported rather than hit later.
#
# The Debian packages are Debian's own; only unoserver has no package, and it
# has to see the "uno" module that python3-uno installs system-wide, which is
# what --system-site-packages is for.
#
# Two of the pins end in "*" rather than a rebuild number. Debian rebuilds a
# package per architecture without touching its source, and the counter it
# bumps is not in step across them: pngquant is +b1 on amd64 and +b2 on
# aarch64, unpaper +b2 and +b3. One literal version cannot satisfy both, so
# these pin the Debian revision and let the rebuild number float. Renovate's
# deb manager reads amd64 only and would pin an aarch64 build to a version
# that does not exist there, which is exactly the breakage; its pattern does
# not match a "*", so it leaves these two alone.
#
# NGINX faces Ingress and the network; Stirling PDF itself only listens on
# loopback.
ARG STIRLING_PDF_VERSION="2.14.3"
ARG UNOSERVER_VERSION="3.7"
ARG WEASYPRINT_VERSION="70.0"
# hadolint ignore=DL3008,DL3013
RUN \
    apt-get update \
    \
    && apt-get install -y --no-install-recommends \
        fonts-crosextra-caladea=20200211-2 \
        fonts-crosextra-carlito=20230309-2 \
        fonts-dejavu=2.37-8 \
        fonts-liberation2=1:2.1.5-3 \
        fonts-noto-core=20201225-2 \
        fontforge-nox=1:20230101~dfsg-4+b1 \
        ghostscript=10.05.1~dfsg-1+deb13u1 \
        imagemagick=8:7.1.1.43+dfsg1-1+deb13u11 \
        libreoffice-calc-nogui=4:25.2.3-2+deb13u6 \
        libreoffice-draw-nogui=4:25.2.3-2+deb13u6 \
        libreoffice-impress-nogui=4:25.2.3-2+deb13u6 \
        libreoffice-java-common=4:25.2.3-2+deb13u6 \
        libreoffice-writer-nogui=4:25.2.3-2+deb13u6 \
        nginx=1.26.3-3+deb13u8 \
        ocrmypdf=16.7.0+dfsg1-3 \
        openjdk-25-jre-headless=25.0.4.1+1-1~deb13u1 \
        pngquant=2.18.0-1* \
        poppler-utils=25.03.0-5+deb13u4 \
        python3-uno=4:25.2.3-2+deb13u6 \
        python3-venv=3.13.5-1 \
        qpdf=12.2.0-1 \
        tesseract-ocr-eng=1:4.1.0-2 \
        tesseract-ocr-osd=1:4.1.0-2 \
        tesseract-ocr=5.5.0-1+b1 \
        unpaper=7.0.0-3* \
    \
    && python3 -m venv --system-site-packages /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir --prefer-binary \
        "opencv-python-headless" \
        "pdf2image" \
        "unoserver==${UNOSERVER_VERSION}" \
        "weasyprint==${WEASYPRINT_VERSION}" \
    \
    && mkdir -p /opt/stirling-pdf \
    && curl -J -L -o /opt/stirling-pdf/stirling-pdf.jar \
        "https://github.com/Stirling-Tools/Stirling-PDF/releases/download/v${STIRLING_PDF_VERSION}/Stirling-PDF.jar" \
    \
    && rm -f -r /etc/nginx \
    \
    && find /opt/venv \
        \( \
            \( -type d -a -name __pycache__ \) \
            -o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
        \) \
        -exec rm -rf '{}' + \
    \
    && apt-get clean \
    && rm -f -r \
        /tmp/* \
        /usr/share/doc/* \
        /usr/share/man/* \
        /var/{cache,log}/* \
        /var/lib/apt/lists/*

# Copy root filesystem
COPY rootfs /

# Health check
HEALTHCHECK \
    --start-period=5m \
    CMD curl --silent --fail http://127.0.0.1:48080/api/v1/info/status || exit 1

# Build arguments
ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_DESCRIPTION
ARG BUILD_NAME
ARG BUILD_REF
ARG BUILD_REPOSITORY
ARG BUILD_VERSION

# Labels
LABEL \
    io.hass.name="${BUILD_NAME}" \
    io.hass.description="${BUILD_DESCRIPTION}" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version=${BUILD_VERSION} \
    maintainer="Franck Nijhof <opensource@frenck.dev>" \
    org.opencontainers.image.title="${BUILD_NAME}" \
    org.opencontainers.image.description="${BUILD_DESCRIPTION}" \
    org.opencontainers.image.vendor="Home Assistant Community Apps" \
    org.opencontainers.image.authors="Franck Nijhof <opensource@frenck.dev>" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.url="https://frenck.dev/home-assistant-apps" \
    org.opencontainers.image.source="https://github.com/${BUILD_REPOSITORY}" \
    org.opencontainers.image.documentation="https://github.com/${BUILD_REPOSITORY}/blob/main/README.md" \
    org.opencontainers.image.created=${BUILD_DATE} \
    org.opencontainers.image.revision=${BUILD_REF} \
    org.opencontainers.image.version=${BUILD_VERSION}
