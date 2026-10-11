
# Stage 1: Build the React application
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 3000
# If Nginx default is port 80, you can either map ECS container port to 80 or reconfigure nginx to listen on 3000
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
