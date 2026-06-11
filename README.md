# ChatApp-part-3---complete
##Student information
Khanyani Mnwana
ST10489373
-----------------------
##Project Information
Part 3 introduced 9 static ArrayLists instead of the single list from Part 2. They're static so one shared copy exists across every Message object in the session.
They split into two groups:
The "all processed" group tracks every message that was either sent or stored — never disregarded. These four lists stay perfectly index-aligned with each other at all times:
------------------------
messageIDs — the ID of each processed message
messageHashes — the hash of each processed message
recipientList — the recipient of each processed message
allMessageTexts — the text of each processed message
--------------------------
The "sent only" group tracks only messages the user chose to Send (option 1). These three lists are index-aligned with each other:

sentMessages — text of sent messages
sentHashes — hashes of sent messages
sentRecipients — recipients of sent messages
-----------------------------
Then the standalone lists:

disregardedMessages — text only, never searched or displayed
storedMessages — populated by reading the JSON file at startup, not by the user clicking Store
storedThisSession — an integer counter (not a list) that increments when the user stores a message this session
-------------------------------
The reason for the split is that messageIDs, messageHashes, recipientList, and allMessageTexts grow for both Send and Store. If you tried to use sentMessages as your lookup target for search or delete, the indices would fall out of sync the moment any stored message appeared. The "all processed" group stays perfectly aligned for searching, while the "sent only" group stays perfectly aligned for the report.
-------------------------------
sentMessage() — populating the right lists
When option 1 (Send) is chosen, the message text, hash, ID, and recipient get added to both groups — the "sent only" group and the "all processed" group.
When option 2 (Disregard) is chosen, only disregardedMessages gets the text. Nothing else is tracked.
When option 3 (Store) is chosen, the hash, ID, recipient, and text go into the "all processed" group only — not sentMessages. The storedThisSession counter increments by 1, and storeMessage() writes the JSON line to the file.
--------------------------------
loadStoredMessages() — reading the JSON file
Called once from MainApp right after the user successfully logs in. It opens messages.json with a BufferedReader, reads it line by line, parses each line as a JSONObject using the org.json library, extracts the "message" field, and adds it to the storedMessages list. If the file doesn't exist yet (first run), the IOException is caught and the app continues normally without crashing.
This is why storedMessages is separate from the "all processed" group — it represents messages from previous sessions, loaded back in at startup.
--------------------------------
displayLongestMessage()
Loops through storedMessages (the JSON-loaded list), tracks the longest string found, and returns it. Uses storedMessages specifically because the POE requirement says the longest message feature searches stored messages, not sent ones.
--------------------------------
searchByMessageID()
Loops through messageIDs. When a match is found at index i, it returns allMessageTexts.get(i). Because both lists are in the "all processed" group, the index is always valid regardless of whether the message was sent or stored.
--------------------------------
searchByRecipient()
Loops through recipientList. Every index where the recipient matches gets allMessageTexts.get(i) appended to a StringBuilder. Returns all matches because a recipient can appear more than once.
--------------------------------
deleteByHash()
Loops through messageHashes. When the hash matches at index i, it grabs the text from allMessageTexts.get(i) for the success message, then removes index i from all four "all processed" lists. It then uses sentHashes.indexOf(hash) to find the correct position within the "sent only" group and removes from those three lists too if found. The indexOf lookup is used rather than index i directly, because the "all processed" index and the "sent only" index will differ whenever any stored messages exist between sent ones.
---------------------------------
printMessages() — the report
Loops through sentMessages using an integer index. At each index it reads from sentHashes.get(i) and sentRecipients.get(i) to get the matching hash and recipient. This is safe because all three "sent only" lists are always the same length and always aligned with each other.
--------------------------------
returnTotalMessages()
Returns sentMessages.size() + storedThisSession. It uses the counter rather than storedMessages.size() because storedMessages is populated from the JSON file at startup — it would count messages from previous sessions, not just the current one.
---------------------------------
clearMessages() and addToStoredMessages()
Both are test helpers. clearMessages() resets all 9 lists and the counter to zero before each @Test runs, preventing state from leaking between tests. addToStoredMessages() lets tests put entries into storedMessages directly without needing a real messages.json file on disk.
------------------------------------
MainApp.java — three additions
Message.loadStoredMessages() is called once right after isLoggedIn becomes true, before the menu loop starts. Option 4 was added to the menu display and the switch statement. The storedMessagesMenu() method handles the sub-menu with options a through f plus q to return, each calling the appropriate Message static method.
The readInt() helper was also added — it wraps every input.nextInt() call in a try/catch so a user typing a letter can't throw an InputMismatchException and crash the app.

MessageTest.java — the six new tests
Each test calls Message.clearMessages() in @BeforeEach so no state carries over. The five POE messages are created fresh before every test using the test constructor that accepts a fixed messageID, making hashes predictable.
testSentMessagesArray_correctlyPopulated — sends messages 1 and 4, then checks the report contains both texts.
testDisplayLongestMessage_returnsCorrectMessage — loads all five POE texts into storedMessages via addToStoredMessages(), then asserts the longest one is returned.
testSearchByMessageID_returnsCorrectMessage — sends messages 1 and 4, searches by message 4's known ID "1000000004", expects "It is dinner time!".
testSearchByRecipient_returnsAllMatchingMessages — sends all five messages, searches for +27838884567 (messages 2 and 5's recipient), asserts both texts appear in the result.
testDeleteByHash_removesCorrectMessage — sends message 2, gets its hash, calls deleteByHash(), asserts the exact success string including the message text.
testDisplayReport_containsRequiredFields — sends messages 1 and 2, asserts the report contains the hash, recipient, and text for each.
