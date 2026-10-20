# Render image: the official LibreChat release plus this repo's librechat.yaml.
# Pulling the prebuilt image avoids building the frontend on Render, which needs far more than 512 MB.
FROM registry.librechat.ai/librechat-ai/librechat:v0.8.8

COPY --chown=node:node librechat.yaml /app/librechat.yaml

ENV NODE_ENV=production \
    HOST=0.0.0.0 \
    PORT=3080

EXPOSE 3080

# Run node directly (no npm wrapper) and cap the heap to fit the 512 MB free instance.
CMD ["node", "--max-old-space-size=384", "api/server/index.js"]
