Running ssh -i /Users/ruzic/.ssh/ruzicl ec2-user@34.212.76.65

Had to modify the ebcli_installer.py in _install_ebcli def:
    # added by Luka                                                                                                                                                           
    # https://github.com/aws/aws-elastic-beanstalk-cli-setup/issues/148#issuecomment-1676717954                                                                               
    install_args.append('--no-build-isolation')
    returncode = _exec_cmd(install_args, quiet)

Had to fix bug: https://github.com/onnela-lab/beiwe-backend/issues/319

Per Eli’s instructions after, change these to: … @sentry.io/1234567

Fire-base configuration -- that is done via Beiwe developers
However, Luka has keys he will provide.  This enables surveys to be taken on phones. 
https://github.com/onnela-lab/beiwe-backend/wiki/Firebase-Configuration
