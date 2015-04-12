# How to prepare coreos
set TERM

    infocmp | ssh -T -p 2222 core@127.0.0.1 'cat > "$TERM.info" && tic "$TERM.info"'

copy files

    tar -c unit-files | ssh -C -p 2222 core@127.0.0.1 "tar -x"

then, from the coreos fleetmanager

    fleetctl start unit-files/instances/*
