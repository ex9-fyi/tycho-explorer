# Build stage
FROM oven/bun:latest as builder
WORKDIR /app

# Accept build arguments for Vite environment variables
ARG VITE_WEBSOCKET_URL
ARG VITE_WEBSOCKET_URL_ETHEREUM
ARG VITE_WEBSOCKET_URL_BASE
ARG VITE_WEBSOCKET_URL_UNICHAIN

# Set them as environment variables for the build
ENV VITE_WEBSOCKET_URL=$VITE_WEBSOCKET_URL
ENV VITE_WEBSOCKET_URL_ETHEREUM=$VITE_WEBSOCKET_URL_ETHEREUM
ENV VITE_WEBSOCKET_URL_BASE=$VITE_WEBSOCKET_URL_BASE
ENV VITE_WEBSOCKET_URL_UNICHAIN=$VITE_WEBSOCKET_URL_UNICHAIN

# Log the build process
RUN echo "🔵 [BUILD] Starting frontend build process..."

# Copy package files
COPY package.json bun.lock ./
RUN echo "🔵 [BUILD] Installing dependencies with bun..."
RUN bun install --frozen-lockfile || (echo "🔴 [ERROR] Failed to install dependencies" && exit 1)

# Copy source code
COPY . .
RUN echo "🔵 [BUILD] Building production bundle..."
RUN bun run build || (echo "🔴 [ERROR] Build failed" && exit 1)
RUN echo "🟢 [BUILD] Frontend build completed successfully"

# Production stage
FROM nginx:alpine
RUN echo "🔵 [PROD] Setting up nginx for production..."

# Install curl for health checks
RUN apk add --no-cache curl

# Copy built files
COPY --from=builder /app/dist /usr/share/nginx/html
RUN echo "🟢 [PROD] Static files copied to nginx directory"

# Copy nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf
RUN echo "🟢 [PROD] Nginx configuration applied"

EXPOSE 80

# nginx:alpine already has a proper entrypoint, just use it
CMD ["nginx", "-g", "daemon off;"]