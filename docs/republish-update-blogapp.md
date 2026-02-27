  <!-- Never edit the published files on the server directly. The published files are compiled binaries (DLLs) that you can't    
   meaningfully modify. -->

  Proper workflow for updates:

  1. Edit your source code in C:\BizfirstAI-Blog\ (the cloned repo)

  2. Test locally - run and verify your changes work

  3. Publish the app from your local machine:
  dotnet publish -c Release -o ./publish

  4. Upload to server (from local machine):
  scp -r ./publish/* your-username@your-server-ip:/tmp/blog-update/

  5. Deploy on server:
  sudo systemctl stop blog.service
  sudo rm -rf /var/www/blog/* && sudo mv /tmp/blog-update/* /var/www/blog/
  sudo chown -R blogapp:blogapp /var/www/blog
  sudo systemctl start blog.service

  <!-- This is exactly what's described in the "Updating the App Later" section at the bottom of your deployment guide
  (docs/quickstart-deploy.md:156-165).

  The published version on the server is just the compiled output - think of it as read-only production files that get      
  replaced with each deployment. -->

