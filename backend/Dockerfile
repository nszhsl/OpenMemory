# ===== BUILD STAGE =====
FROM node:20-alpine AS builder

WORKDIR /app

# --- Mirrors (CN friendly) ---
ARG ALPINE_MIRROR="https://mirrors.aliyun.com/alpine"
ARG NPM_REGISTRY="https://registry.npmmirror.com"
ARG NODE_DISTURL="https://npmmirror.com/mirrors/node"

# 1) Switch Alpine repositories to a CN mirror (reduces apk timeout)
RUN sed -i "s#https\?://dl-cdn.alpinelinux.org/alpine#${ALPINE_MIRROR}#g" /etc/apk/repositories \
 && apk add --no-cache python3 make g++ curl bash

# 2) Use CN-friendly registries for npm & bun (bun install reads npm-style env)
ENV npm_config_registry="${NPM_REGISTRY}" \
    NPM_CONFIG_REGISTRY="${NPM_REGISTRY}" \
    npm_config_disturl="${NODE_DISTURL}" \
    NPM_CONFIG_DISTURL="${NODE_DISTURL}" \
    # node-gyp headers mirror (important for native addons)
    npm_config_nodedir="" \
    # Optional: reduce noisy network retries
    npm_config_fetch_retries=3 \
    npm_config_fetch_retry_maxtimeout=60000 \
    npm_config_fetch_timeout=60000

# (Optional, but very useful for some packages in CN)
# If you use any of these deps, uncomment what you need.
# ENV ELECTRON_MIRROR="https://npmmirror.com/mirrors/electron/" \
#     SASS_BINARY_SITE="https://npmmirror.com/mirrors/node-sass/" \
#     SHARP_DIST_BASE_URL="https://npmmirror.com/mirrors/sharp-libvips/" \
#     PLAYWRIGHT_DOWNLOAD_HOST="https://npmmirror.com/mirrors/playwright/" \
#     PUPPETEER_DOWNLOAD_HOST="https://npmmirror.com/mirrors/chromium-browser-snapshots/" \
#     CYPRESS_DOWNLOAD_MIRROR="https://npmmirror.com/mirrors/cypress/" \
#     PUPPETEER_SKIP_DOWNLOAD="true"

# 3) Install Bun (you already use bun.net.cn which is CN-friendly)
RUN curl -fsSL https://bun.net.cn/install | bash \
 && mv /root/.bun /opt/bun \
 && ln -s /opt/bun/bin/bun /usr/local/bin/bun
ENV PATH="/opt/bun/bin:${PATH}"

# 4) Copy manifests + lockfile(s) (important for reproducible builds)
COPY package*.json ./
# If you have bun.lockb / package-lock.json, copy them too (no harm if absent in repo)
COPY bun.lockb* ./
COPY package-lock.json* ./

# 5) Install deps (bun will use npm_config_registry)
RUN bun install --frozen-lockfile

# 6) Build
COPY src/ ./src/
COPY tsconfig.json ./
RUN bun run build

# 7) Prune dev deps to reduce final size
RUN npm prune --omit=dev


# ===== PRODUCTION STAGE =====
FROM node:20-alpine AS production

WORKDIR /app

ARG ALPINE_MIRROR="https://mirrors.aliyun.com/alpine"

# Switch Alpine repositories + tzdata
RUN sed -i "s#https\?://dl-cdn.alpinelinux.org/alpine#${ALPINE_MIRROR}#g" /etc/apk/repositories \
 && apk add --no-cache tzdata

# Timezone (CN typical)
ENV TZ="Asia/Shanghai"

# Create non-root user
RUN addgroup -g 1001 -S appgroup \
 && adduser -u 1001 -S appuser -G appgroup

# Copy production artifacts
COPY --from=builder /app/node_modules ./node_modules/
COPY --from=builder /app/dist ./dist/
COPY package.json ./

# Persistent data dir + permissions
RUN mkdir -p /data \
 && chown -R appuser:appgroup /data /app \
 && chmod -R go-w /app

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD node -e "require('http').get('http://localhost:8080/health', (res) => process.exit(res.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

ENTRYPOINT ["npm", "start"]
