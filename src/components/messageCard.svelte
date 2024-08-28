<script lang="ts">
    import { addNewMessage } from "$lib/messageRoutines";
    import { AllChatMessages, type ChatMessage } from "../stores/messageStore";
    import MessageItem from "./messageItem.svelte";      
    import { onMount, tick } from 'svelte';
    import * as signalR from '@microsoft/signalr';
    

    let msgList:any;            //  the message list contianer div we display
    let msgText:string;         //  the new message we add from the user input
    let signalrHubUrl:string = 'http://67.171.50.230:333/danochat' //  user for the signalRHub
    let connection:signalR.HubConnection = null;  //  the signalr connection  
    let hasBeenInited:boolean  = false;
    
    //onMount(()=>{scrollToBottom(msgList)})    
    
    async function saveNewMessage() 
    {
        if(connection == null || connection.state != signalR.HubConnectionState.Connected)
        {
            alert("Please connect to the hub first");
            return;
        }        
        
        let newChatText = msgText.trim();
        if(newChatText.length  < 1)
        {
            return; 
        }
        await connection.invoke('SendMessage', 'SvelteDano',newChatText);
            
        msgText = '';
        await tick();   
        await scrollToBottom(msgList);        
    }
    
    const scrollToBottom = async (node:any) => {
      await  node.scroll({ top: 1-node.scrollHeight, behavior: 'smooth' });
    };  

    const signalRHubUrl = async (e:any) => {
      console.log(e.target.files[0]);      
     }
   
    const connnectToHub = async () => 
    {
        console.log("connnectToHub");
        signalrHubUrl =  signalrHubUrl.trim();
    
        if(signalrHubUrl.length < 1)
        {
            alert("Please enter a hub url");
            return;
        }
        
        connection =  new signalR.HubConnectionBuilder()
            .withUrl(signalrHubUrl)
            .configureLogging(signalR.LogLevel.Information)   
            .build();
        
        await connection.start()
        
        //
        //  Check if the methods have been registered for the connectionhub.On("message")
        //  according to the docs a connection.off("JoinChat") should remove the message handler
        //  appears not to be true, so we need to prevent the connection.on('???') as we will get 
        //  multiple events for the same message, as each successive register will get adn event.  
        //  sendMesage once but connet to hub a second time, as the time out is a few minutes, upon 
        //  re-connetiong to the hub as one would need to... We would be registering a second 
        //  message handler thus get 2, 3, more once for each reconnet we are using here...  While 
        //  limited to only our client, other clients connected will only see the one comming back 
        //  from the server message getting broadcasted.
        //
        
        if( !hasBeenInited)
        {    
            hasBeenInited=true;        
            // console.log('connection started');
            connection.off('JoinChat');        
            connection.on('JoinChat', (message:string) => {
                console.log(`Joined room ${message}`);
                addNewMessage(`Joined room ${message}`);
                return ;
            });
            connection.off('ReceiveMessage');
            connection.on('ReceiveMessage', (message:string) => {
                console.log(message);
                addNewMessage(message);
                return ;
            }); 
            connection.off('SendMessage');
            connection.on('SendMessage', (userName:String,message:string) => {
                console.log(`${message}`);
                addNewMessage(`${message}`);
                return ;
            });
        }
        
        await connection.invoke('JoinChat', 'Dano');
        
    }
    
</script> 

<div class="mesagesSection">
    
    <div bind:this={msgList}>
        {#each $AllChatMessages as item}
        <MessageItem bind:msg={item}/> 
        {/each}
    </div> 
</div>
    
<div class="newMessageInput">    
    <div class="newMessage"><label for="newMessageText">Message</label></div>
    <div class="newMessage"><input type="text" bind:value={msgText} id="newMessageText">
        <button class="addMessage" on:click={()=>saveNewMessage()}>Add Message</button>
    </div> 
    <label  class="newMessage" for="hubUrl"  on:change={signalRHubUrl}>SignalR Hub Url&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</label>
    <button class="addMessage" on:click={()=>connnectToHub()}>Connect to Hub</button>
    
    <input  class="newMessage" type="text" bind:value={signalrHubUrl} id="hubUrl" on:change={(e)=>signalRHubUrl(e)}/>   
</div>
 
<style>
   
    .mesagesSection
    {
      display: flex; 
      align-items: left;      
      flex-direction: column;   
      background-color: #ecffff;
      color: rgba(44, 44, 252, 0.974)   ;        
      border-radius: 7px;
      border:3px double blue; 
      height: 333px;
      width: 60%; 
      flex: 1;
      overflow-y: auto;        
    }   
    
    .newMessageInput
    {
        margin-top: 20px;
        width: 60%;    
        height: 50%;
        background-color: #ecffff;
        color: rgba(44, 44, 252, 0.974)   ;        
        border-radius: 7px;
        border: double 3px blue;
    }
    
    .addMessage 
    {
        margin:  10px 6px;
    }
    
    .newMessage     
    {
        margin-top:  5px;
        margin-left: 7px;
        margin-bottom: 5px ; 
        width: 303px;
    }
    
</style>


