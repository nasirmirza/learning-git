****
Keep AI initiative in the background.
****

Hi, My name is Nasir Mirza and I am a developer in Liquidity and Margin Services group. I would like to give a brief introduction to the AI agent we created in Eliza for processing SWIFT Settlement Instructions. We have been live in production for over two weeks with our agent and lets go over to see what it does for our users.
****
Keep Phoenix image in the background.
****

SWIFT SSI stands for Standing Settlement Instructions. They are exchanged between financial institutions to route and settle funds or securities accurately.
Phoenix is the central hub and repository for SSIs used across different LOBs in Markets. 
We currently hold cash and securities instructions for FX, Liquidity, Margin and Global collateral management applications. We ingest SSIs from DTCC and SWIFT network for our FX clients.
We have developed Eliza SSI Agent to automate the processing of free format SWIFT messages like MT199, MT299, MT999 that contain settlement instruction details. 
****
Samples (spend 20 – 30 seconds)
******
Here are some sample messages that that are sent by financial institutions to update our copy of SSIs. Its an unstructured format and had to be read by OPS users manually to analyze and update downstream systems. 
Note: talk about 2 messages. Signle CHF and multiple and show how language differs.
****
FLOW DIAGRAM
******
Here is the high-level design of processing free-format SWIFT messages using SSI Agent.
Step 1: The process begins when a SWIFT message is received by EMX (Enterprise Message Exchange) from the SWIFT network. Our listener subscribes for incoming messages (MT199, MT299, or MT671).
%%% Pause
Step 2: We then perform a client lookup in UCM to check if the client referenced in the message is valid. 
Step 3: If the message is free-format then Phoenix will process the message by sending it to Eliza SSI agent. The SSI Agent is configured and trained to extract settlement instruction data from the free format text. 
%%% Pause
Explain the step3 free-format text and output
The data extracted from the LLM will be in a structured JSON format which will be stored in our repository. 
%%% Play the rest of flow
Step 4: FX OPS users can view these messages in the SSI Hub Portal interface. They see a side-by-side view of the raw message and the extraction done by the agent. The details are updated in Wall Street Systems (WSS) for post-trade settlement activity.

NEXEN
Lets see some live data processed by our agent over the past two weeks. 



