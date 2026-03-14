# Install Redis on Ubuntu

Redis is an in-memory key–value database commonly used for caching, queues, sessions, and real-time analytics.

Step 1: Update system packages
sudo apt update
sudo apt upgrade -y


# Step 2: Install Redis

Ubuntu repositories already contain Redis.

sudo apt install redis-server -y
Step 3: Check Redis version
redis-server --version

Example output:

Redis server v=7.x.x sha=...

# Step 4: Start Redis service

sudo systemctl start redis
sudo systemctl start redis-server

# Step 5: Enable Redis on boot

sudo systemctl enable redis-server


# Step 6: Verify Redis is running

sudo systemctl status redis

Or test using Redis CLI:

redis-cli ping

Output:

PONG