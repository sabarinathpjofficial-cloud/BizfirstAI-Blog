I need help deploying my .NET 8 Blog application to a production Ubuntu server.

Context:
- The app is a Blogifier-based .NET 8 project
- I already ran: dotnet publish -c Release -o publish
- My publish folder is ready locally
- Server OS: Ubuntu 22.04
- I have SSH access
- I want to host it at: blog.mycompany.com
- I want to use:
    - Kestrel
    - systemd service
    - Nginx reverse proxy
    - HTTPS with Let's Encrypt

Please give me:

1. Step-by-step server setup instructions
2. Install .NET runtime on Ubuntu
3. Create /var/www/blog folder
4. Upload and run the published app
5. Create systemd service for auto-start
6. Configure Nginx reverse proxy
7. Configure SSL using Certbot
8. Commands to restart and check logs
9. Basic firewall setup (ufw)

Please provide:
- Exact commands
- Configuration file examples
- Folder paths
- Service file content
- Nginx config content
- Production best practices

Assume I am deploying for the first time.