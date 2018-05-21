# Prime-Ubuntu-18.04
Nvidia Prime without rebooting

# Dependencies:
bbswitch\
Nvidia 390\
Ubuntu 18.04 (might work with other distros if you change some paths)

# How to build:
First build the rust part with cargo\
then:\
cd src\
make install\
sudo systemctl enable prime-socket\
sudo systemctl start prime-socket\
