# Single Dockerfile for both API and Worker services
FROM node:20-alpine3.20

# Create working directory
WORKDIR /app

# Optimize layer caching: copy dependency manifests first with correct ownership
COPY --chown=node:node package*.json ./

# Install production dependencies only and clear npm cache to keep image small
RUN npm install --omit=dev --no-audit --no-fund && npm cache clean --force

# Copy application source code with correct ownership (avoids duplicate layer)
COPY --chown=node:node . .

# Switch to non-root user
USER node

# Expose API port
EXPOSE 3000

# Health check for the API service
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget -qO- http://127.0.0.1:3000/api/health || exit 1

# Default command for API
CMD ["npm", "start"]
