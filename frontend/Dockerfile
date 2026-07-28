FROM nginx:alpine

# Copy frontend static files
COPY . /usr/share/nginx/html/

# Copy custom Nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose HTTP port
EXPOSE 80

# Start Nginx in the foreground
# will not run in the background
CMD ["nginx", "-g", "daemon off;"]  

