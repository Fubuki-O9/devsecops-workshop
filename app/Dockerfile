FROM node:18-alpine

# Create a non-root user to run the app (matches k8s securityContext runAsUser 1001)
RUN addgroup -g 1001 -S nodejs && adduser -u 1001 -S appuser -G nodejs

WORKDIR /app

# Install dependencies first (better layer caching). Use package.json, not a
# hardcoded package name. `npm ci` needs a lockfile; fall back to `npm install`.
COPY package*.json ./
RUN npm install --omit=dev && npm cache clean --force

# Copy the rest of the source
COPY . .

# Drop to the non-root user
USER appuser

EXPOSE 3000

CMD ["node", "index.js"]
