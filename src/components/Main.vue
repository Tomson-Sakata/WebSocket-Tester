<template>
<!-- 
xs(Extra Small)・・・主にスマホ対応
sm(Small)・・・主にタブレット
md(Medium) ・・・主にノートパソコンや横表示のタブレット
lg(Large)・・・主にデスクトップ
xl(Extra large)・・・4Kや大きなモニター 


v-if="$vuetify.breakpoint.xs"
-->
    <v-container fluid>
        <v-row dense>
            <v-col>
                <v-toolbar flat dense>
                <v-chip :class="stateStyle" label style="width:100%">{{stateMessage}}</v-chip>
                </v-toolbar>
            </v-col>
        </v-row>
        <v-row dense>
            <v-col>
                <v-toolbar flat dense>
                    <v-text-field
                        v-model="url"
                        hide-details
                        outlined
                        dense
                        prefix="URL:"
                        class="mr-3" />
                    <v-btn class="mr-3 primary" @click="onClickConnect" :disabled="state != 'disconnect' && state != 'error'">
                        <v-icon>mdi-lan-connect</v-icon>
                        <span v-if="!$vuetify.breakpoint.xs">connect</span>
                    </v-btn>
                    <v-btn class="primary" @click="onClickDisconnect" :disabled="state != 'connected'">
                        <v-icon>mdi-lan-disconnect</v-icon>
                        <span v-if="!$vuetify.breakpoint.xs">disconnect</span>
                    </v-btn>
                </v-toolbar>
            </v-col>
        </v-row>
        <v-row dense>
            <v-col>
                <v-expansion-panels flat multiple accordion>
                    <v-expansion-panel>
                        <v-expansion-panel-header><span class="title">SEND</span></v-expansion-panel-header>
                        <v-expansion-panel-content>
                            <v-card outlined>
                                <v-card-text>
                                    <v-textarea
                                        v-model="sendMessage"
                                        hide-details
                                        outlined
                                        dense
                                        prefix="message:"
                                        class="mr-3"
                                        clearable
                                        auto-grow
                                        :append-icon="state == 'connected' && sendMessage != null && sendMessage.length > 0 ? 'mdi-send' : ''"
                                        @click:append="onClickSendMessage" />
                                    <v-simple-table dense fixed-header height="150" class="mt-3">
                                        <thead>
                                            <tr>
                                                <th class="text-left">date time</th>
                                                <th class="text-left">message</th>
                                            </tr>
                                        </thead>
                                        <tbody>
                                            <v-scroll-x-transition v-for="(message, index) in sentMessages" :key="index">
                                                <tr>
                                                    <td class="text-left" style="white-space:nowrap">{{message.dateTime}}</td>
                                                    <td class="text-left" style="width:100%">{{message.message}}</td>
                                                </tr>
                                            </v-scroll-x-transition>
                                        </tbody>
                                    </v-simple-table>
                                    <v-btn class="primary mt-3" @click="onClickClearSentHistory">history clear</v-btn>
                                </v-card-text>
                            </v-card>
                        </v-expansion-panel-content>
                    </v-expansion-panel>
                    <v-expansion-panel>
                        <v-expansion-panel-header><span class="title">RECEIVE</span></v-expansion-panel-header>
                        <v-expansion-panel-content>
                            <v-card outlined>
                                <v-card-text>
                                    <v-simple-table dense fixed-header height="150">
                                        <thead>
                                            <tr>
                                                <th class="text-left">date time</th>
                                                <th class="text-left">message</th>
                                            </tr>
                                        </thead>
                                        <tbody>
                                            <v-scroll-x-transition v-for="(message, index) in receivedMessages" :key="index">
                                                <tr >
                                                    <td class="text-left" style="white-space:nowrap">{{message.dateTime}}</td>
                                                    <td class="text-left" style="width:100%">{{message.message}}</td>
                                                </tr>
                                            </v-scroll-x-transition>
                                        </tbody>
                                    </v-simple-table>
                                    <v-btn class="primary mt-3" @click="onClickClearReceivedHistory">history clear</v-btn>
                                </v-card-text>
                            </v-card>
                        </v-expansion-panel-content>
                    </v-expansion-panel>
                </v-expansion-panels>
            </v-col>
        </v-row>
    </v-container>
</template>

<script>
    class Tools {
        static getCurrentDateTime () {
            const date = new Date()
            return date.getFullYear() + "-" + ("0"+(date.getMonth() + 1)).slice(-2) + "-" + ("0" + date.getDate()).slice(-2) + " " +
                   ("0" + date.getHours()).slice(-2) + ":" + ("0" + date.getMinutes()).slice(-2) + ":" + ("0" + date.getSeconds()).slice(-2)
        }
    }
    export default {
        computed: {
            stateMessage: function () {
                switch (this.state) {
                    case "disconnect": return "Not connected to the server. Enter the URL and click the Connect button to connect."
                    case "connecting": return "Connecting to the server... please wait."
                    case "connected": return "Connected to the server"
                    case "disconnect_wait_reconnect": return "Disconnected from the server. Waiting for reconnect time."
                    case "error": return "An error has occurred."
                    default: return "unknown state"
                }
            },
            stateStyle: function () {
                switch (this.state) {
                    case "disconnect": return "warning"
                    case "connecting": return "warning"
                    case "connected": return "primary"
                    case "disconnect_wait_reconnect": return "warning"
                    case "error": return "red white--text"
                    default: return "primary"
                }
            }
        },
        data: function () {
            return {
                state: "disconnect",
                url: "ws://192.168.1.4:10000",
                socket: null,
                connected: false,
                sendMessage: "",
                sentMessages: [],
                receivedMessages: [],
                lastError: null
            }
        },
        methods: {
            onClickConnect: function () {
                const self = this
                const socketCreator = function () {
                    self.setState("connecting")
                    self.socket = new WebSocket(self.url)

                    self.socket.onopen = function (e) {
                        self.connected = true
                        self.setState("connected")
                    }
                    self.socket.onmessage = function (e) {
                        console.log("onmessage", e.data)
                        self.onReceived(e)
                    }
                    self.socket.onclose = function (e) {
                        if (self.connected) {
                            self.setState("disconnect_wait_reconnect")
                            setTimeout(function(){
                                socketCreator()
                            }, 5000)
                        }
                    }
                    self.socket.onerror = function (e) {
                        console.log("onerror", e)
                        self.lastError = e
                        self.setState("error")
                    }
                }
                socketCreator()
            },
            onClickDisconnect: function () {
                this.socket.close()
                this.socket = null
                this.connected = false
                this.setState("disconnect")
            },
            onClickSendMessage: function () {
                this.socket.send(this.sendMessage)
                console.log("sent message", this.sendMessage)
                this.sentMessages.splice(
                    0,
                    0,
                    {
                        message: this.sendMessage,
                        dateTime: Tools.getCurrentDateTime()
                    }
                )
            },
            onClickClearSentHistory: function () {
                this.sentMessages = []
            },
            onClickClearReceivedHistory: function () {
                this.receivedMessages = []
            },
            onReceived: function (e) {
                this.receivedMessages.splice(
                    0,
                    0,
                    {
                        raw: e,
                        message: e.data.toString(),
                        dateTime: Tools.getCurrentDateTime()
                    }
                )
            },
            setState: function (state) {
                this.state = state
            }
        }
    }
</script>
