ARG BUILD_FROM
FROM $BUILD_FROM as base

ENV LANG C.UTF-8
ARG BUILD_VERSION

RUN apk add --no-cache socat tini nodejs eudev && \
    # Validation to confirm that curl is installed in the base image.
    curl --version || (echo "curl missing" && exit 1)

# Dependencies and build
FROM base as dependencies_and_build

RUN apk add --no-cache --virtual .buildtools npm make gcc g++ linux-headers udev git python3 && \
    if [ "${BUILD_VERSION}" = "edge" ]; \
    then \
        echo "Installing Edge version" && \
        git clone -b dev --single-branch --depth 1 \
        https://github.com/Koenkk/zigbee2mqtt.git /app && \
        mkdir /app/dist && \
        jq -n --arg commit $(eval cd /app;git rev-parse --short HEAD) '$commit' > /app/dist/.hash ; \
    else \
        echo "Installing Stable version ${BUILD_VERSION%-*}" && \
        curl -sL -o "/app.tar.gz" \
        "https://github.com/Koenkk/zigbee2mqtt/archive/${BUILD_VERSION%-*}.tar.gz" && \
        tar xzvf "/app.tar.gz"  && rm "/app.tar.gz" && \
        mv -v "zigbee2mqtt-${BUILD_VERSION%-*}" app && \
        mkdir /app/dist && \
        jq -n --arg commit ${BUILD_VERSION%-*} '$commit' > /app/dist/.hash ; \
    fi; \
    echo "Installed Zigbee2MQTT @ version $(cat /app/dist/.hash)" && \
    cd /app && \
    npm install -g $(jq -r '.packageManager' package.json) && \
    pnpm install --frozen-lockfile --no-optional --unsafe-perm && \
    pnpm run build && \
    rm -rf node_modules && \
    pnpm install --frozen-lockfile --no-optional --unsafe-perm --prod && \
    # force serialport to rebuild on current platform instead of using prebuilds
    rm -rf `find ./node_modules/.pnpm/ -wholename "*/@serialport/bindings-cpp/prebuilds" -type d` && \
    pnpm rebuild @serialport/bindings-cpp

# Custom zigbee-herdsman-converters with Halo smoke detector support
FROM base as custom_converters

ENV HUSKY=0

RUN apk add --no-cache --virtual .buildtools npm make gcc g++ linux-headers git python3 && \
    git clone -b feature/halo-smoke-detector --single-branch --depth 1 \
        https://github.com/alex-savin/zigbee-herdsman-converters.git /custom-converters && \
    cd /custom-converters && \
    npm install -g $(jq -r '.packageManager' package.json) && \
    pnpm install --frozen-lockfile --no-optional --unsafe-perm && \
    pnpm run build && \
    rm -rf node_modules && \
    pnpm install --frozen-lockfile --no-optional --unsafe-perm --prod --ignore-scripts

# Replace stock zigbee-herdsman-converters with custom build
FROM dependencies_and_build as patched

RUN CONV_PKG=$(find /app/node_modules/.pnpm -maxdepth 1 -type d -name "zigbee-herdsman-converters@*" | head -1) && \
    if [ -n "$CONV_PKG" ]; then \
        CONV_DIR="$CONV_PKG/node_modules/zigbee-herdsman-converters"; \
        echo "Replacing stock converters at $CONV_DIR"; \
        rm -rf "$CONV_DIR/dist" "$CONV_DIR/node_modules"; \
    else \
        CONV_DIR="/app/node_modules/zigbee-herdsman-converters"; \
        echo "Replacing stock converters at $CONV_DIR (flat layout)"; \
        rm -rf "$CONV_DIR/dist" "$CONV_DIR/node_modules"; \
    fi && \
    echo "$CONV_DIR" > /tmp/conv_dir_path

COPY --from=custom_converters /custom-converters/dist /tmp/custom-conv-dist
COPY --from=custom_converters /custom-converters/node_modules /tmp/custom-conv-node_modules
COPY --from=custom_converters /custom-converters/package.json /tmp/custom-conv-package.json

RUN CONV_DIR=$(cat /tmp/conv_dir_path) && \
    cp -r /tmp/custom-conv-dist "$CONV_DIR/dist" && \
    cp -r /tmp/custom-conv-node_modules "$CONV_DIR/node_modules" && \
    cp /tmp/custom-conv-package.json "$CONV_DIR/package.json" && \
    echo "Custom zigbee-herdsman-converters (Halo) installed at $CONV_DIR" && \
    rm -rf /tmp/custom-conv-dist /tmp/custom-conv-node_modules /tmp/custom-conv-package.json /tmp/conv_dir_path

# Release
FROM base as release

WORKDIR /app

COPY rootfs /
COPY --from=patched /app/node_modules ./node_modules
COPY --from=patched /app/dist ./dist
COPY --from=patched /app/package.json /app/LICENSE /app/index.js ./

ENV NODE_ENV production

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --start-interval=5s \
  CMD curl -s http://localhost:8092/ > /dev/null || exit 1

ENTRYPOINT [ "/sbin/tini", "--", "/docker-entrypoint.sh"]
