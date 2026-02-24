# Setup SystemD task for your NodeJs Project

Setting up a SystemD script for your Node.js backend is a professional-grade approach to ensuring your services remain stable, especially for a mission-critical platform. While Docker handles container orchestration, SystemD acts as the primary manager for the host system's processes.

## Key Benefits of SystemD

- Automatic Restarts: If your Node.js process crashes due to an unhandled exception or memory leak, SystemD will automatically restart it within seconds.

- Boot-Time Start: It ensures your services launch automatically as soon as the industrial gateway or server powers on, which is vital for maintaining 24/7 reliability.

- Logging Integration: It pipes all console.log and console.error outputs directly to journald, providing a centralized location for debugging system health.

- Resource Control: You can set limits on CPU and Memory usage, preventing your backend from starving other critical system processes on the host server.

- Dependency Management: You can configure the script to wait until the network or MongoDB is fully active before attempting to start the service.

## Commands

Start a service

```
sudo systemctl start <Service Name>

```

Stop a service

```
sudo systemctl stop <Service Name>

```

Restart a service

```
sudo systemctl restart <Service Name>

```
