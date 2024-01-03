
## The Purpose of Source Code Signing

It can prevent files from being tampered with by hackers during the transfer process.

## How to Use xLua's Signature Feature

* Use Tools/KeyPairsGen.exe to generate a public-private key pair. The key_ras file contains the private key, and key_ras.pub contains the public key. Please safeguard these files carefully. The private key is crucial for game security, so ensure it is kept confidential;
* Use Tools/FilesSignature.exe to sign the source code:
  * Place the key_ras file in the execution directory;
  * The parameters are the source and target directories. This tool will automatically sign all files with a .lua extension in the source directory and its subdirectories, placing them in the target directory while maintaining the same directory structure;
* Wrap your existing CustomLoader with SignatureLoader for use;
  * SignatureLoader has two constructors, one takes the public key, which is the content found in key_ras.pub, and the other takes the original Loader;
