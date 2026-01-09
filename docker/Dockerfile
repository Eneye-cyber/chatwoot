############################################
# Base Node Image (Chatwoot-compatible)
############################################
FROM node:20-bookworm AS node

############################################
# Pre-builder stage
############################################
FROM ruby:3.4.4-bookworm AS pre-builder

# ------------------------
# Versions (LOCKED)
# ------------------------
ARG NODE_VERSION=20.11.1
ARG PNPM_VERSION=8.15.5
ARG BUNDLER_VERSION=2.5.11

# ------------------------
# Rails config
# ------------------------
ARG RAILS_ENV=production
ENV RAILS_ENV=${RAILS_ENV}
ENV RAILS_SERVE_STATIC_FILES=true
ENV BUNDLE_WITHOUT="development:test"
ENV BUNDLE_PATH="/gems"

# ------------------------
# Low-memory safe defaults
# ------------------------
ENV NODE_OPTIONS="--max-old-space-size=2048"
ENV VITE_CJS_IGNORE_WARNING=true

# ------------------------
# System deps
# ------------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
  build-essential \
  git \
  curl \
  libpq-dev \
  imagemagick \
  libvips \
  tzdata \
  && rm -rf /var/lib/apt/lists/*

# ------------------------
# Ruby setup
# ------------------------
RUN gem install bundler -v ${BUNDLER_VERSION}

# ------------------------
# Node + PNPM
# ------------------------
COPY --from=node /usr/local/bin/node /usr/local/bin/
COPY --from=node /usr/local/lib/node_modules /usr/local/lib/node_modules

RUN ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm \
 && ln -s /usr/local/lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx \
 && npm install -g pnpm@${PNPM_VERSION}

WORKDIR /app

# ------------------------
# Install Ruby deps
# ------------------------
COPY Gemfile Gemfile.lock ./
RUN bundle config set without 'development test' \
 && bundle install -j 2 --retry 3

# ------------------------
# Install JS deps
# ------------------------
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# ------------------------
# App source
# ------------------------
COPY . .

RUN mkdir -p log tmp

############################################
# OPTIONAL: Asset Precompile
############################################
ARG PRECOMPILE_ASSETS=true

RUN if [ "$RAILS_ENV" = "production" ] && [ "$PRECOMPILE_ASSETS" = "true" ]; then \
  SECRET_KEY_BASE=precompile_placeholder \
  RAILS_LOG_TO_STDOUT=enabled \
  bundle exec rake assets:precompile; \
  fi

# Cleanup (save memory + image size)
RUN rm -rf node_modules tmp/cache spec

############################################
# Runtime stage
############################################
FROM ruby:3.4.4-bookworm

ARG RAILS_ENV=production
ENV RAILS_ENV=${RAILS_ENV}
ENV BUNDLE_PATH="/gems"
ENV RAILS_SERVE_STATIC_FILES=true

RUN apt-get update && apt-get install -y --no-install-recommends \
  libpq5 \
  imagemagick \
  libvips \
  tzdata \
  && rm -rf /var/lib/apt/lists/* \
  && gem install bundler -v 2.5.11

COPY --from=node /usr/local/bin/node /usr/local/bin/
COPY --from=node /usr/local/lib/node_modules /usr/local/lib/node_modules

COPY --from=pre-builder /gems /gems
COPY --from=pre-builder /app /app

WORKDIR /app
EXPOSE 3000
