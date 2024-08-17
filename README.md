# objectstreaming
In Python AI server, it often receives/sends client query in streaming, e.g., to receive a client query in text/audio stream format, and after AI server processes the query and sends test/audio streams back to client. This protocol implementation converts object or wav object into byte sequence to be streamed between client and server, and to be used in a typical use case like this:

<p align='center'>
    <img width="754" alt="Client and Server Streaming" src="https://github.com/user-attachments/assets/0488971e-3664-436b-9ef8-05dc08ff674f">
</p>

The client can be based on any codebase, as long as client and server follows the same protocol as shown in the figure below. The first 4 bytes are the total length of this byte sequence, and second 4 bytes are the length of your customized header object (in this case the header is TALAudio object), the following bytes in header length are the header object itself,  and the remaining bytes are the audio bytes.

<p align='center'>
    <img width="754" alt="Bytes protocol" src="https://github.com/user-attachments/assets/09acc6c8-2f1b-48f6-a76f-faedcf3f3ea5">
</p>

# wav streaming:
typical use case is client side send audio wav (as a query) to LLM backend server, which receives the audio and use ASR to convert wav into text query, process the query, convert text answer into wav using TTS, and sends the wav back to the client.

sample code:
```python
from wav_streaming import TALAudio, tal_encoder

#helper function

def audio_response_to_stream(audio_iter):
    r = TALAudio()  
    order = 0
    for audio in audio_iter:
        #the example of autio_iter is an iterator of [str, int, int, nd.array]
        text, bit_depth, sample_rate, channels, audio_array = audio # str, int, int, nd.array
        if len(audio_array)==0:
            continue
        # keep answer
        answer += text
        order += 1
        r.order = order
        r.sign = 1 if order==1 else 2 #start if order is 1, else 2 meaning the middle message
        r.bitDepth = bit_depth
        r.audioType = f"int{bit_depth}"
        r.sampleRate = sample_rate
        r.channels = channels
        r.answer = text
        r.question = query
        wav = audio_array.tobytes() # nd.array to bytes
        yield tal_encoder(r, wav=wav)

audio_iter = #somewhere in your code you get the iterator of audios
#get the bytes
rs = audio_response_to_stream(audio_iter)
#fastapi streaming back to client use application/octet-stream mimetype
return StreamingResponse(rs, media_type='application/octet-stream')
```

# object streaming:
typical use case is client side send text query to LLM backend server in fastapi for example, which process the query, and answer back to client in stream.

sample code:
```
from object_streaming import AnbJsonStreamCoder

#helper function
async def convert_results_to_stream():
  for i in range(3):
    #a header
    r_header = {
      'status': 'success',
      'reason': 'none',
    }
    #an answer back to client
    r_dict = {
      'answer': f'LLM answers here, {i}...',
      'source': 'documentX_page_12',
    }
    #convert to bytes
    r_bytes = AnbJsonStreamCoder.encode(r_header,r_obj)
    yield r_bytes

#get the bytes
rs = convert_results_to_stream()
#fastapi streaming back to client use application/octet-stream mimetype
return StreamingResponse(rs, media_type='application/octet-stream')
```
