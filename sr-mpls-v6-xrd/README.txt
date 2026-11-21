# Don't forget you need these sysctl's for XRd

sysctl -w fs.inotify.max_user_instances=64000
sysctl -w fs.inotify.max_user_watches=64000

