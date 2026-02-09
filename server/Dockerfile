# Use lightweight Node image
FROM node:18-alpine

# Create app directory
WORKDIR /app

# Copy package files first (better caching)
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy rest of the source code
COPY . .

# Create uploads directory (important for your code)
RUN mkdir -p uploads

# Expose backend port
EXPOSE 5000

# Start the server
CMD ["node", "index.js"]
