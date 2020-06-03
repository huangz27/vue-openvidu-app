<template>

    <div class="container">
       <!--  <div class="row justify-content-center">
            <div class="col-md-8">
                <div class="card">
                    <div class="card-header">Vue Axios Post - ItSolutionStuff.com</div>
                    <div class="card-body">
                        <strong>Output:</strong>
                        <pre>

                        {{output}}

                        </pre>
                    </div>
                </div>
            </div>
        </div>
<button type="button" class="btn btn-primary" @click="createSession">Test REST API (return session ID)</button>
<button type="button" class="btn btn-primary" @click="createToken">Test REST API (return token)</button>
<button type="button" class="btn btn-primary" @click="joinSession">Join </button> -->

        <div id="join" v-show="!joined">
            <h1>Join a video session</h1>
            <form @submit.prevent="joinSession"> 
                <p>
                    <label>Session:</label>
                    <input type="text" v-model="sessionId" required>
                </p>
                <p>
                    <input type="submit" value="JOIN">
                </p>
            </form>
        </div>

        <div id="session" v-show="joined">
            <h1 v-text="sessionId"></h1>
            <input type="button" @click="leaveSession" value="LEAVE">
            <button type="button" class="btn btn-primary" @click="start_record">Record Session </button>
            <button type="button" class="btn btn-primary" @click="stop_record">Stop Recording </button>
            <div >
                <div id ="publisher" ><h3>YOU</h3></div>
                <div id ="subscriber" ><h3>OTHERS</h3></div>
            </div>
            <div>
            <video ref="videoRef" src="" id="video-container" width="30%" controls></video>
            </div>
        </div>
        
 
    </div>
    
</template>

     

<script>
import { OpenVidu } from 'openvidu-browser';

const OPENVIDU_SERVER_URL = 'https://' + window.location.hostname + ':4443';
const OPENVIDU_SERVER_SECRET = 'MY_SECRET';
var OV;
var session;
    export default {

        mounted() {
            console.log('Component mounted.')
            this.$refs.videoRef.src = "https://localhost:4443/recordings/sessionA/sessionA.mp4";
        },

        data() {
            return {
              joined: false,
              sessionId: "sessionA",
              recordingId: "",
              timer: ""
            };
        },

        methods: {
            joinSession() {
            //joinSession(e) {
              //e.preventDefault(); //either use submit.prevent or pass event 
              
              OV = new OpenVidu();
              session = OV.initSession();

              session.on("streamCreated", function (event) {
                  session.subscribe(event.stream, "subscriber");
              });

              this.getToken(this.sessionId).then(token => {

              session.connect(token)
                .then(() => {
                    this.joined = true;
                    var resolution_data = (window.innerWidth * 0.4) + "x";  //first half of the resolution is enough
                    var publisher = OV.initPublisher("publisher", { resolution: resolution_data});
                    session.publish(publisher);
                })
                .catch(error => {
                    console.log("There was an error connecting to the session:", error.code, error.message);
                    });
            });
                // this.timer = setInterval(this.restartRecording, 3000)
            },

            leaveSession() {
                session.disconnect();
                this.stop_record();
                this.joined = false;
            },

            

            getToken(mySessionId) {
                return this.createSession(mySessionId).then((sessionId) => this.createToken(sessionId));
            },

            createSession(sessionId) {
                return new Promise((resolve, reject) => {
                
                let currentObj = this;
                this.axios({
                    method:'post', 
                    url: OPENVIDU_SERVER_URL + "/api/sessions",
                    data: JSON.stringify({ customSessionId: sessionId }),
                    //data: {"customSessionId": "sessionId" }, 
                    // auth: {
                    //         username: "OPENVIDUAPP",
                    //         password: "MY_SECRET",
                    //     },
                    headers: {
                        "Authorization": "Basic " + btoa("OPENVIDUAPP:" + OPENVIDU_SERVER_SECRET),
                        "Content-Type": "application/json"
                    },
                     
                })
                .then(function (response) {
                    currentObj.output = response.data.id; // for debugging
                    console.log('CREATE SESION', response);
                    resolve(response.data.id)
                })
                .catch(function (response) {

                    
                    var error = Object.assign({}, response);
                    currentObj.output = error.response;  // for debugging
                     
                    if (error.response.status === 409) {
                        resolve(sessionId); //sessionId
                    } else {
                        console.log(error);
                        console.warn(
                            'No connection to OpenVidu Server. This may be a certificate error at ' +
                            OPENVIDU_SERVER_URL,
                        );
                        if (
                            window.confirm(
                                'No connection to OpenVidu Server. This may be a certificate error at "' +
                                OPENVIDU_SERVER_URL +
                                '"\n\nClick OK to navigate and accept it. ' +
                                'If no certificate warning is shown, then check that your OpenVidu Server is up and running at "' +
                                OPENVIDU_SERVER_URL +
                                '"',
                            )
                        ) {
                            window.location.assign(OPENVIDU_SERVER_URL + '/accept-certificate');
                        }
                    }

                    reject(error)  //not sure of the purpose
                });
            });
            },

            createToken(sessionId) {
                return new Promise((resolve, reject) => {
                
                let currentObj = this;
                this.axios({
                    method:'post', 
                    url: OPENVIDU_SERVER_URL + "/api/tokens",
                    data: JSON.stringify({ session: sessionId }),
                    //data: {"session": "sessionId" }, 
                    // auth: {
                    //         username: "OPENVIDUAPP",
                    //         password: "MY_SECRET",
                    //     },
                    headers: {
                        "Authorization": "Basic " + btoa("OPENVIDUAPP:" + OPENVIDU_SERVER_SECRET),
                        "Content-Type": "application/json"
                    },
                     
                })
                .then(function (response) {
                    currentObj.output = response.data.token; // for debugging
                    console.log('TOKEN', response);
                    resolve(response.data.id)
                })
                .catch (function (error) {
                    currentObj.output = error; // for debugging
                    reject(error)
                });
            });
            },

            start_record() {
                this.axios({
                    method:'post', 
                    url: OPENVIDU_SERVER_URL + "/api/recordings/start",
                    data: JSON.stringify({ 
                        session: this.sessionId , 
                        outputMode: "COMPOSED",  //outputmode must be set to composed to change resolution
                        resolution: "640x480" }), //default is 1920x1080
                    headers: {
                        "Authorization": "Basic " + btoa("OPENVIDUAPP:" + OPENVIDU_SERVER_SECRET),
                        "Content-Type": "application/json"
                    },
                     
                })
                .then(response => {
                    this.recordingId = response.data.id;
                    console.log("starting recording: " + this.recordingId);
                })
                .catch(error => {
                    var newerror = Object.assign({}, error);
                     console.log(newerror.status);
                });
            },
            stop_record(){

                this.axios({
                    method:'post', 
                    url: OPENVIDU_SERVER_URL + "/api/recordings/stop/" + this.recordingId,
                    //data: JSON.stringify({ session: this.sessionId }),
                    headers: {
                        "Authorization": "Basic " + btoa("OPENVIDUAPP:" + OPENVIDU_SERVER_SECRET),
                        "Content-Type": "application/json"
                    },
                     
                })
                .then(response => {
                    console.log("stopped recording:" + response.data.id);
                })
                .catch (error => {
                    var newerror = Object.assign({}, error);
                     console.log(newerror.status);
                });
            },

            // restartRecording(){
            //     this.stop_record();
            //     this.start_record();
            // }
        }

    }


</script>

<style scoped>
#publisher {    
    float: left;
    margin: 10px;
    width: 40%;
}

#subscriber {
    float: right;
    margin: 10px;
    width: 40%;
}
</style>

