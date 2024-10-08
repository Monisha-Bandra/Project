<!-- chat/templates/chat/room.html -->

{% extends 'base.html' %}

{% block title %}Chat Room - {{ room_name }}{% endblock %}

{% block content %}
    {% if user.is_authenticated %}
        Hello, <b>{{ user.username|capfirst|default:'Guest' }}!</b>
        {% if user.user_type == 'pro' %}
            (Pro)<sup>*</sup>
        {% else %}
            (Basic)<sup>*</sup>
        {% endif %}
        <br>
        <span style="color: #999; font-size: 0.8em;"><sup>*</sup>Pro users can create groups and add users, in addition to seeing entire chat history.</span><br><br>
        
        <b>Group :</b> {{room_name|capfirst}}<br>
        <b>Participants</b>
        <ul id="participant-list">
            {% for participant in participants %}
                <li>
                    <a href="{% url 'chat:direct_messages' %}?receiver={{ participant }}">{{ participant|capfirst }}</a>
                    {% if user.user_type == 'pro' and participant != request.user %}
                        <form action="{% url 'chat:remove_user_from_room' room_name participant.username %}" method="POST" style="display:inline;" onsubmit="return confirmDelete();">
                            {% csrf_token %}
                            <input type="submit" value="X" style="color: red; background: none; border: none; cursor: pointer; font-size: 12px;">
                        </form>
                    {% endif %}
                </li>
            {% empty %}
                <li>No other participants.</li>
            {% endfor %}
        </ul>

        {% if user.user_type == 'pro' %}
        <b>Invite</b> User to the group.
        <form method="POST" action="{% url 'chat:invite_to_room' room_name %}">
            {% csrf_token %}
            <input type="text" name="username" placeholder="Enter username to invite" required>
            <input type="submit" value="Invite">
        </form>
        {% endif %}
        {% if messages %}
            <br>
                {% for message in messages %}
                    {{ message }}<br>
                {% endfor %}
        {% endif %}
        <br>

        <div>
            <div id="chat-log" style="border: 1px solid #ccc; padding: 10px; height: 300px; overflow-y: auto;"></div><br>
            <div class="chat-input-container">
                <input id="chat-message-input" type="text" size="50">
                <button id="emoji-button" style="font-size: 1.0em;">😊</button>      
                <input id="chat-message-submit" type="button" value="Send">
            </div>
        </div>
        <div id="emoji-picker" style="display: none; border: 1px solid #ccc; padding: 10px; position: absolute; background: white; z-index: 10;">
            <span class="emoji" data-emoji="😀">😀</span>
            <span class="emoji" data-emoji="😂">😂</span>
            <span class="emoji" data-emoji="❤️">❤️</span>
            <span class="emoji" data-emoji="👍">👍</span>
            <span class="emoji" data-emoji="🎉">🎉</span>
            <span class="emoji" data-emoji="😎">😎</span>
            <span class="emoji" data-emoji="🥳">🥳</span>
            <span class="emoji" data-emoji="😢">😢</span>
            <span class="emoji" data-emoji="🔥">🔥</span>
            <span class="emoji" data-emoji="🎈">🎈</span>
        </div>

        {{ room_name|json_script:"room-name" }}
        <a href="{% url 'users:dashboard' %}">Dashboard</a>
        <a href="#" onclick="logout()">Logout</a>&nbsp;
    {% else %}
        <h5>You can access this page only if you are logged in.</h5>
        <a href="{% url 'users:dashboard' %}">Dashboard</a>
    {% endif %}
    
    <!-- Message Template -->
    <div id="message-template" style="display: none;">
        <span class="timestamp" style="font-style: italic; color: grey; font-size: 0.8em;"></span>  <b class="username"></b> : <span class="message-content"></span>
        <button class="delete-message-button" style="color: red; background: none; border: none; cursor: pointer;">x</button>
    </div>

    <script>
        const roomName = JSON.parse(document.getElementById('room-name').textContent);
        let chatSocket = null;  // Store the WebSocket connection reference

        function connectToChat() {
            if(!chatSocket) {
                chatSocket = new WebSocket(
                    'ws://'
                    + window.location.host
                    + '/ws/chat/'
                    + roomName
                    + '/'
                );
            }

            chatSocket.onmessage = function(e) {
                const data = JSON.parse(e.data);
                // Check if messages are present
                console.log(data)
                if (data.messages) {
                    // Clear existing messages so when a bulk list of messages are sent, clients see only the latest messages
                    document.getElementById('chat-log').innerHTML = '';  // Set innerHTML to empty string to clear all children

                    data.messages.forEach(msg => {
                        const { username, message, timestamp, message_id } = msg;
                        appendMessage(username, message, timestamp, message_id);
                    });

                    // Show notification for new message
                   // showNotification("", "a message is deleted");

                } else {
                    console.log('onmessage appendMessage for a single message')
                    const { username = "Unknown User", message = "No message", timestamp = "Unknown time", message_id } = data;
                    
                    appendMessage(username, message, timestamp, message_id);
                    
                    // Show notification for new message
                    showNotification(username, message);
                }

                function showNotification(username, message) {
                console.log('showNotification: ', Notification.permission)
                if (Notification.permission === 'granted') {
                    const notification = new Notification(`New message from ${username}`, {
                        body: message,
                        icon: '' // Customize this path if needed
                    });
                }
        }
            };

            chatSocket.onclose = function(e) {
                console.error('Chat socket closed. Reconnecting...');
                chatSocket = null;  // Clear the reference
                setTimeout(connectToChat, 2000); // Reconnect after 2 seconds
            };
        }

        function appendMessage(username, message, timestamp, message_id) {
            const messageTemplate = document.getElementById('message-template').cloneNode(true);
            messageTemplate.id = '';
            messageTemplate.style.display = '';
            messageTemplate.querySelector('.timestamp').textContent = timestamp;
            messageTemplate.querySelector('.username').textContent = username;
            messageTemplate.querySelector('.message-content').textContent = message;

            if ('{{ user.user_type }}' === 'pro') {
                const deleteButton = messageTemplate.querySelector('.delete-message-button');
                deleteButton.dataset.messageId = message_id;
                deleteButton.addEventListener('click', () => handleDeleteMessage(message_id));
            } else {
                messageTemplate.querySelector('.delete-message-button').remove();
            }

            document.getElementById('chat-log').appendChild(messageTemplate);
            scrollToBottom();
        }

        

        // Request notification permission
        if (Notification.permission === 'default') {
            Notification.requestPermission().then(permission => {
                if (permission === 'granted') {
                    console.log('Notification permission granted.');
                }
            });
        }

        function scrollToBottom() {
            const chatLog = document.getElementById('chat-log');
            chatLog.scrollTop = chatLog.scrollHeight;
        }

        function handleDeleteMessage(messageId) {
            if (confirm("Are you sure you want to delete this message?")) {
                // Send a delete message through the WebSocket
                chatSocket.send(JSON.stringify({
                    type: 'delete',
                    message_id: messageId
                }));
            }
        }

        connectToChat();
        
        document.querySelector('#chat-message-input').focus();
        document.querySelector('#chat-message-input').onkeyup = function(e) {
            if (e.keyCode === 13) {  // enter, return
                document.querySelector('#chat-message-submit').click();
            }
        };

        document.querySelector('#chat-message-submit').onclick = function(e) {
            const messageInputDom = document.querySelector('#chat-message-input');
            var messageInput = messageInputDom.value.trim();
            if (messageInput) {
                chatSocket.send(JSON.stringify({
                    message: messageInput,
                    username: '{{ request.user.username|capfirst }}',
                    type: 'message',
                }));
                messageInputDom.value = '';
            }
        };
        
        function logout() {
            // Create a form element
            var form = document.createElement('form');
            form.method = 'POST';
            form.action = '{% url "users:logout" %}';
        
            // Create a hidden input field for the CSRF token
            var csrfTokenInput = document.createElement('input');
            csrfTokenInput.type = 'hidden';
            csrfTokenInput.name = 'csrfmiddlewaretoken';
            csrfTokenInput.value = '{{ csrf_token }}';
        
            // Append the CSRF token input to the form
            form.appendChild(csrfTokenInput);
        
            // Append the form to the document body
            document.body.appendChild(form);
        
            // Submit the form
            form.submit();
        }
    
        function confirmDelete() {
            return confirm("Are you sure you want to remove this user from the group?");
        }

            //emoji picker
        document.getElementById('emoji-button').onclick = function() {
            const emojiPicker = document.getElementById('emoji-picker');
            emojiPicker.style.display = emojiPicker.style.display === 'none' ? 'block' : 'none';
        };

        document.querySelectorAll('.emoji').forEach(function(emoji) {
            emoji.onclick = function() {
                const emojiValue = emoji.getAttribute('data-emoji');
                const chatInput = document.getElementById('chat-message-input');
                chatInput.value += emojiValue;  // Append the selected emoji to the input
                chatInput.focus();  // Refocus on the input field
              //  document.getElementById('emoji-picker').style.display = 'none';  // Hide the picker after selection
            };
        });

        // Close the emoji picker if clicking outside of it
        document.addEventListener('click', function(event) {
            const emojiPicker = document.getElementById('emoji-picker');
            const emojiButton = document.getElementById('emoji-button');
            if (!emojiPicker.contains(event.target) && event.target !== emojiButton) {
                emojiPicker.style.display = 'none';
            }
        });
    </script>
    
{% endblock %}




{% extends 'base.html' %}

{% block content %}
    {% if user.is_authenticated %}
        Hello, <b>{{ user.username|capfirst|default:'Guest' }}!</b>
        {% if user.user_type == 'pro' %}
            (Pro)<sup>*</sup>
        {% else %}
            (Basic)<sup>*</sup>
        {% endif %}
        <br>
        <span style="color: #999; font-size: 0.8em;"><sup>*</sup>Pro users can delete messages and see entire chat history.</span><br><br>

        {% if current_receiver %}
            messaging with: <b>{{ current_receiver.username|capfirst }}</b><br><br>
            <div>
                <div id="message-log" style="border: 1px solid #ccc; padding: 10px; height: 300px; overflow-y: auto;"></div><br>
                <div class="message-input-container">
                    <input id="message-input" type="text" size="50">
                    <button id="emoji-button" style="font-size: 1.0em;">😊</button>
                    <input id="message-submit" type="button" value="Send">
                </div>
            </div>
            <div id="emoji-picker" style="display: none; border: 1px solid #ccc; padding: 10px; position: absolute; background: white; z-index: 10;">
                <span class="emoji" data-emoji="😀">😀</span>
                <span class="emoji" data-emoji="😂">😂</span>
                <span class="emoji" data-emoji="❤️">❤️</span>
                <span class="emoji" data-emoji="👍">👍</span>
                <span class="emoji" data-emoji="🎉">🎉</span>
                <span class="emoji" data-emoji="😎">😎</span>
                <span class="emoji" data-emoji="🥳">🥳</span>
                <span class="emoji" data-emoji="😢">😢</span>
                <span class="emoji" data-emoji="🔥">🔥</span>
                <span class="emoji" data-emoji="🎈">🎈</span>
            </div>
        {% endif %}

        <!-- Message Template -->
        <div id="message-template" style="display: none;">
            <span class="timestamp" style="font-style: italic; color: grey; font-size: 0.8em;"></span>  <b class="username"></b> : <span class="message-content"></span>
            <button class="delete-message-button" style="color: red; background: none; border: none; cursor: pointer;">x</button>
        </div>

        <a href="#" onclick="logout()">Logout</a>&nbsp;
        <a href="{% url 'users:dashboard' %}">Dashboard</a>
    {% else %}
        <h5>You can access this page only if you are logged in.</h5>
        <a href="{% url 'users:dashboard' %}">Dashboard</a>
    {% endif %}

<script>
    const messageLog = document.querySelector('#message-log'); // Declare messageLog globally
    const receiverSelect = document.querySelector('#receiver-select');
    let currentReceiver = '{{ current_receiver.username }}';

    // WebSocket connection
    const receiver = '{{ current_receiver.username }}'; 
    let chatSocket = null;  // Store the WebSocket connection reference
    
    function connectToChat() {
        if(!chatSocket) {
            chatSocket = new WebSocket(
                'ws://' + window.location.host + '/ws/direct_messages/' + currentReceiver + '/'
            );
        }

        chatSocket.onmessage = function(e) {
            const data = JSON.parse(e.data);
            console.log(data.messages)
            // Check if messages are present
            console.log(data)
            if (data.messages) {
                // Clear existing messages so when a bulk list of messages are sent, clients see only the latest messages
                document.getElementById('message-log').innerHTML = '';  // Set innerHTML to empty string to clear all children

                data.messages.forEach(msg => {
                    const { timestamp, username, message, message_id } = msg;
                    appendMessage(timestamp, username, message, message_id);
                });
                //showNotification("", "A message is deleted")
            } else {
                console.log('onmessage appendMessage for a single message')
                const { message = "", sender = "Unknown User", timestamp = "Unknown time", message_id } = data;
                
                appendMessage(timestamp, sender, message, message_id);
                //showNotification(sender, message)
            }

            function showNotification(username, message) {
            console.log('showNotification: ', Notification.permission)
            if (Notification.permission === 'granted') {
                const notification = new Notification(`New message from ${username}`, {
                    body: message,
                    icon: '' // Customize this path if needed
                });
            }
    }
        };

        chatSocket.onclose = function(e) {
            console.error('Chat socket closed unexpectedly');
        };
    }
    
    function appendMessage(timestamp, username, message, message_id) {
        const capitalizedUsername = username.charAt(0).toUpperCase() + username.slice(1);
        const messageTemplate = document.getElementById('message-template').cloneNode(true);

        messageTemplate.id = '';
        messageTemplate.style.display = '';
        messageTemplate.querySelector('.timestamp').textContent = timestamp;
        messageTemplate.querySelector('.username').textContent = capitalizedUsername;
        messageTemplate.querySelector('.message-content').textContent = message;

        if ('{{ user.user_type }}' === 'pro') {
            const deleteButton = messageTemplate.querySelector('.delete-message-button');
            deleteButton.dataset.messageId = message_id;
            deleteButton.addEventListener('click', () => handleDeleteMessage(message_id));
        } else {
            messageTemplate.querySelector('.delete-message-button').remove();
        }

        document.getElementById('message-log').appendChild(messageTemplate);
        scrollToBottom();
    }


    // Request notification permission
    if (Notification.permission === 'default') {
        Notification.requestPermission().then(permission => {
            if (permission === 'granted') {
                console.log('Notification permission granted.');
            }
        });
    }

    function scrollToBottom() {
        const chatLog = document.getElementById('message-log');
        chatLog.scrollTop = chatLog.scrollHeight;
    }

    function handleDeleteMessage(messageId) {
        if (confirm("Are you sure you want to delete this message?")) {
            // Send a delete message through the WebSocket
            chatSocket.send(JSON.stringify({
                type: 'delete',
                message_id: messageId
            }));
        }
    }

    connectToChat();

    document.querySelector('#message-input').focus();
    document.querySelector('#message-input').onkeyup = function(e) {
        if (e.keyCode === 13) {  // enter, return
            document.querySelector('#message-submit').click();
        }
    };

    document.querySelector('#message-submit').onclick = function(e) {
        const messageInputDom = document.querySelector('#message-input');
        var messageInput = messageInputDom.value.trim();
        if (messageInput) {
            chatSocket.send(JSON.stringify({
                message: messageInput,
                type:'message',
                'receiver': currentReceiver,
            }));
            messageInputDom.value = '';
        }
    };

    function logout() {
        // Create a form element
        var form = document.createElement('form');
        form.method = 'POST';
        form.action = '{% url "users:logout" %}';
    
        // Create a hidden input field for the CSRF token
        var csrfTokenInput = document.createElement('input');
        csrfTokenInput.type = 'hidden';
        csrfTokenInput.name = 'csrfmiddlewaretoken';
        csrfTokenInput.value = '{{ csrf_token }}';
    
        // Append the CSRF token input to the form
        form.appendChild(csrfTokenInput);
    
        // Append the form to the document body
        document.body.appendChild(form);
    
        // Submit the form
        form.submit();
    }

    //emoji picker
    document.getElementById('emoji-button').onclick = function() {
        const emojiPicker = document.getElementById('emoji-picker');
        emojiPicker.style.display = emojiPicker.style.display === 'none' ? 'block' : 'none';
    };

    document.querySelectorAll('.emoji').forEach(function(emoji) {
        emoji.onclick = function() {
            const emojiValue = emoji.getAttribute('data-emoji');
            const chatInput = document.getElementById('message-input');
            chatInput.value += emojiValue;  // Append the selected emoji to the input
            chatInput.focus();  // Refocus on the input field
          //  document.getElementById('emoji-picker').style.display = 'none';  // Hide the picker after selection
        };
    });

    // Close the emoji picker if clicking outside of it
    document.addEventListener('click', function(event) {
        const emojiPicker = document.getElementById('emoji-picker');
        const emojiButton = document.getElementById('emoji-button');
        if (!emojiPicker.contains(event.target) && event.target !== emojiButton) {
            emojiPicker.style.display = 'none';
        }
    });
</script>

{% endblock %}
