echo "::set-output name=status::$(if [ $? -eq 0 ]; then echo 'success'; else echo 'failure'; fi)"
