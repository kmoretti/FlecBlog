FROM node:22-alpine

WORKDIR /app

COPY .output /app/.output
COPY theme.json ./

ENV NODE_ENV=production
ENV HOST=0.0.0.0
ENV PORT=3000

EXPOSE 3000

CMD ["node", ".output/server/index.mjs"]
