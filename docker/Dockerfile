FROM node:18-alpine AS builder

WORKDIR /app

COPY app/package.json ./
RUN yarn --silent
COPY app .

RUN yarn build

FROM nginx:1.21.0-alpine AS server

COPY --from=builder /app/dist /usr/share/nginx/html
COPY docker/nginx.conf /etc/nginx/conf.d
RUN rm /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]