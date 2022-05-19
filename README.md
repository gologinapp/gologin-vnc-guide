xvfb also supported but vnc scenario is recommended
## 1. Install Nodejs
## 2. Download Orbita
```bash
wget https://releases-deb-orbita.s3.eu-central-1.amazonaws.com/orbita-browser-latest.tar.gz
tar -xvf orbita-browser-latest.tar.gz
```
now we have downloaded and unpacked orbita, for example here: `/root/orbita-browser/chrome` - this is `executablePath` parameter for gologin script

## 3. VNC server setup
```bash 
apt install xfce4 xfce4-goodies
apt install tigervnc-standalone-server
```

## 4. VNC server password
exec script or just run `vncpasswd` to promt for password:
```bash
#!/bin/sh

myuser="root"
mypasswd="My_VnC_PaSsworD"

mkdir /home/$myuser/.vnc
echo $mypasswd | vncpasswd -f > /home/$myuser/.vnc/passwd
chown -R $myuser:$myuser /home/$myuser/.vnc
chmod 0600 /home/$myuser/.vnc/passwd
```


`nano ~/.vnc/xstartup`

and fill with next content:
```bash
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4 
```

`chmod u+x ~/.vnc/xstartup`

## 5. Run vnc server
`vncserver :2 -name $HOSTNAME -geometry 1024x768 -depth 24 -localhost no`

note :2 - `vncPort` parameter for gologin script


## 6. Check running
`vncserver --list`

should look like 
```
X DISPLAY #	RFB PORT #	PROCESS ID
:2		5902		66709
```

## 7. Create sandbox project
```bash
mkdir /opt/test-gologin
cd /opt/test-gologin
npm init
npm i gologin -s
...
```
## 8. If you run vnc as root
modify gologin run.sh to run as root
`nano /opt/test-gologin/node_modules/gologin/run.sh`

then change it to (just add `--no-sandbox`):

`DISPLAY=:$5 $1 --remote-debugging-port=$2 --no-sandbox --proxy-server=$3 --user-data-dir=$4 --tz=$6 --load-extension=$7 --password-store=basic --lang=en --new-window`


## 9. Start gologin profile
create /opt/test-gologin/index.js with example code (dont forget to change token, profile_id and executablePath):
```javascript
const GoLogin = require('gologin');
const delay = ms => new Promise(res => setTimeout(res, ms));

(async () =>{
    const GL = new GoLogin({
        token: 'YourGologinApiToken',
        profile_id: 'YourGologinProfileId',
	    executablePath: '/root/orbita-browser/chrome',
	    extra_params: ['--no-sandbox'],
        vncPort: 02
    });
    console.log('starting profile..')
    const {status, wsUrl} = await GL.start(); 
    console.log('profile ready wsUrl is', wsUrl)
    const browser = await puppeteer.connect({
        browserWSEndpoint: wsUrl.toString(), 
        ignoreHTTPSErrors: true,
    });

    const page = await browser.newPage();

    const viewPort = GL.getViewPort();    

    await page.setViewport({ width: Math.round(viewPort.width * 0.994), height: Math.round(viewPort.height * 0.92) });
    const session = await page.target().createCDPSession();
    const { windowId } = await session.send('Browser.getWindowForTarget');
    await session.send('Browser.setWindowBounds', { windowId, bounds: viewPort });
    await session.detach();
    console.log('goto google.com')
    await page.goto(`https://google.com`, {waitUntil: 'load', timeout: 0});    
    await delay(5000);
    console.log('done. waiting 10 secs');
    await delay(10000)
    await browser.close();
    await GL.stop();
})()
```

then `node index.js`


