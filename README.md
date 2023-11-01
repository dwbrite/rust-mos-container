requirements:
linux, docker/podman

`docker build base -t rust-mos-base`

`docker run --name temp-container rust-mos-base && docker export temp-container -o squashed/myimage.tar && docker stop temp-container && docker rm temp-container`

`docker build squashed -t rust-mos`
