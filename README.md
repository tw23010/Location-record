<template id="mainPage.html">
    <ons-page id="mainPage">
        <ons-toolbar>
            <div class="center">Smartphone Probe</div>
            <div class="right">
                <ons-toolbar-button onclick="fn.toggleOnline()" id="online-flag">
                    ONLINE
                </ons-toolbar-button>
            </div>                
        </ons-toolbar>
        <ons-list>
            <ons-list-item>
                <label class="left">Project</label>
                <label class="center">
                    <ons-input id="project-name" placeholder="Firebase project"></ons-input>
                </label>
            </ons-list-item>
            <ons-list-item>
                <label class="left">Path</label>
                <label class="center">
                    <ons-input id="path-name" placeholder="Database path"></ons-input>
                </label>
            </ons-list-item>                
            <ons-list-item>
                <label class="left">GPS</label>
                <label class="right" id="gps-status">NULL</label>
            </ons-list-item>
            <ons-list-item>
                <label class="left">Motion</label>
                <label class="right" id="motion-status">NULL</label>
            </ons-list-item>
        </ons-list>
        <ons-button modifier="large" id="start-button" onclick="fn.startAcq()">START</ons-button>
    </ons-page>
</template>

<script>
    let updateFlag = false;
    let onlineFlag = false;
    let prevTick = 0;
    let gpsWatchId = 0;
    let posLatitude = [];
    let posLongitude = [];
    let motionX = [];
    let motionY = [];
    let motionZ = [];        
    let motionXAvg = [];
    let motionYAvg = [];
    let motionZAvg = [];
    let motionXStdev = [];
    let motionYStdev = [];
    let motionZStdev = [];        
    let motionXMin = [];
    let motionYMin = [];
    let motionZMin = [];        
    let motionXMax = [];
    let motionYMax = [];
    let motionZMax = [];
    const arrAvg = arr => arr.reduce((a,b) => a + b, 0) / arr.length;
    const arrStdev = arr => {
        const avg = arrAvg(arr);
        let total = 0;
        for (let i = 0; i < arr.length; i++) {
            total += Math.pow((arr[i] - avg),2);
        }
        return Math.sqrt(total/arr.length);            
    };
    const arrMin = arr => {return Math.min(...arr)};
    const arrMax = arr => {return Math.max(...arr)};

    function submitFirebase(t) {
        const projName = document.getElementById('project-name').value;
        const pathName = document.getElementById('path-name').value;
        let Url = 'https://' + projName + '.firebaseio.com/' + pathName + '.json';            
        let Data = {
            timestamp: t,
            latitude: posLatitude,
            longitude: posLongitude,
            motionXAvg: motionXAvg,
            motionYAvg: motionYAvg,
            motionZAvg: motionZAvg,
            motionXStdev: motionXStdev,
            motionYStdev: motionYStdev,
            motionZStdev: motionZStdev,
            motionXMin: motionXMin,
            motionYMin: motionYMin,   
            motionZMin: motionZMin,                
            motionXMax: motionXMax,
            motionYMax: motionYMax,   
            motionZMax: motionZMax                
        };
        const Params = {
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify(Data),
            method: 'POST'
        };
        console.log(JSON.stringify(Data))
        if (onlineFlag) {
            console.log('Adding to ' + Url);
            document.getElementById('online-flag').innerHTML = 'sending';
            fetch(Url, Params).then(data => { 
                document.getElementById('online-flag').innerHTML = 'OFFLINE';
                return data.json(); 
            }).then(res => { 
                console.log(res); 
            });
        }
    }

    function motionHandler(ev) {
        motionX.push(ev.acceleration.x);
        motionY.push(ev.acceleration.y);
        motionZ.push(ev.acceleration.z); 
    }

    window.fn = {};

    window.fn.startAcq = function () {
        if (!updateFlag) {
            console.log('Start updating');
            // GPS tracking
            gpsWatchId = navigator.geolocation.watchPosition(pos => {
                posLatitude.push(pos.coords.latitude);
                posLongitude.push(pos.coords.longitude);
                maxX = arrMax(motionX);
                maxY = arrMax(motionY);
                maxZ = arrMax(motionZ);
                if (maxX != NaN) {
                    motionXAvg.push(arrAvg(motionX));
                    motionYAvg.push(arrAvg(motionY));
                    motionZAvg.push(arrAvg(motionZ));
                    motionXStdev.push(arrStdev(motionX));
                    motionYStdev.push(arrStdev(motionY));
                    motionZStdev.push(arrStdev(motionZ));                        
                    motionXMin.push(arrMin(motionX));
                    motionYMin.push(arrMin(motionY));
                    motionZMin.push(arrMin(motionZ));                        
                    motionXMax.push(arrMax(motionX));
                    motionYMax.push(arrMax(motionY));
                    motionZMax.push(arrMax(motionZ)); 
                    motionX = [];
                    motionY = [];
                    motionZ = [];                             
                }
                document.getElementById('gps-status').innerHTML = pos.coords.latitude.toString() + ',' + pos.coords.longitude.toString();
                if (maxX != NaN) {
                    document.getElementById('motion-status').innerHTML = maxX.toFixed(2) + ',' + maxY.toFixed(2) + ',' + maxZ.toFixed(2); 
                }
                // Sending data
                if (Date.now() > prevTick + 10000) {
                    const t = new Date(pos.timestamp);
                    submitFirebase(t);
                    posLatitude = [];
                    posLongitude = [];                        
                    motionXAvg = [];
                    motionYAvg = [];
                    motionZAvg = [];
                    motionXStdev = [];
                    motionYStdev = [];
                    motionZStdev = [];
                    motionXMin = [];
                    motionYMin = [];
                    motionZMin = [];                                                
                    motionXMax = [];
                    motionYMax = [];
                    motionZMax = [];
                    prevTick = Date.now();
                }
            }, err => {
                console.log(err);
            },
            options={
                enableHighAccuracy: true,
                timeout: 10000
            });
            // Motion tracking       
            try {
                DeviceMotionEvent.requestPermission().then(response => {
                    if (response == 'granted') {
                        window.addEventListener('devicemotion', motionHandler);                            
                    }                        
                });
            } catch {
                window.addEventListener('devicemotion', motionHandler);
            }
            document.getElementById('start-button').innerText = 'STOP';
            updateFlag = true;
        } else {
            console.log('Stop updating');
            navigator.geolocation.clearWatch(gpsWatchId);
            window.removeEventListener('devicemotion', motionHandler);
            gpsWatchId = 0;
            document.getElementById('start-button').innerText = 'START';                
            updateFlag = false;
        }
    };

    window.fn.toggleOnline = function () {
        if (!onlineFlag) {
            console.log('Start syncing');
            onlineFlag = true;
            document.getElementById('online-flag').innerHTML = 'OFFLINE';
        } else {
            console.log('Stop syncing');
            onlineFlag = false;
            document.getElementById('online-flag').innerHTML = 'ONLINE';
        }            
    };
</script>
