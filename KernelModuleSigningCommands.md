## How to sign modules (in this case for VirtualBox) so they can be loaded into your kernel

Pulled bits of this form a few places. This is done in bash.


    mkdir -p /root/signing_keys
    chmod 700 /root/signing_keys
    cd /root/signing_keys
    
    # Fill in the options below under O, CN, and emailAddress
    cat << EOF > /root/signing_keys/configuration_file.config
    [ req ]
    default_bits = 4096
    distinguished_name = req_distinguished_name
    prompt = no
    string_mask = utf8only
    x509_extensions = myexts
    
    [ req_distinguished_name ]
    O = MAKEITUP
    CN = Organization signing key
    emailAddress = YourEMail@somedomain.com
    
    [ myexts ]
    basicConstraints=critical,CA:FALSE
    keyUsage=digitalSignature
    subjectKeyIdentifier=hash
    authorityKeyIdentifier=keyid
    EOF
    
    openssl req -x509 -new -nodes -utf8 -sha256 -days 36500 -batch -config configuration_file.config -outform DER -out my_signing_key_pub.der -keyout my_signing_key.priv
    
    # This will output the contents of the key in text format so you can read it (-text)
    openssl x509 -inform der -text -noout -in my_signing_key_pub.der
    
    # View the current keys
    keyctl list %:.platform
    keyctl list %:.builtin_trusted_keys
    keyctl list %:.blacklist
    
    # import the key onto your systems keychain
    mokutil --import my_signing_key_pub.der
    
    # mokutil will ask for a password and then you will need to reboot to have it added to enroll the key. Prior to enrollment you may see a message saying they key is in "the enrollment request"
    
    # Show the builtin trusted keys
    keyctl list %:.builtin_trusted_keys
    
    # show the platform keys, which is where your new key should be.
    keyctl list %:.platform
    
    # recompile the drivers
    /sbin/vboxconfig
    
    # Now sign them, change the find command to be the modules you want to sign
    for i in `find /lib/modules -name vbox*` ; do /usr/src/kernels/$(uname -r)/scripts/sign-file sha256 my_signing_key.priv my_signing_key_pub.der $i ; done
    
