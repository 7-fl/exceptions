I added a try-catch to the following clause in the server's receive loop:

```erlang

        {request, Pid , {deallocate, Freq}} ->
            NewFrequencies = 
                try deallocate(Frequencies, Freq)
                catch
                    error:{badmatch, _Val} -> Frequencies
                end,
            Pid ! {reply, ok},
            loop(NewFrequencies, Supervisor);
 ```
 That does nothing when the client tries to deallocate a frequency that it does not own.  
 
 Another approach would be to send an error message to the client:
 
 ```
         {request, Pid , {deallocate, Freq}} -> 
            try deallocate(Frequencies, Freq) of
                NewFrequencies -> 
                    Pid ! {reply, ok},
                    loop(NewFrequencies, Supervisor)
            catch
                error:{badmatch, _Val} ->
                    Pid ! {reply, {error, not_allocated}},
                    loop(Frequencies, Supervisor)
            end;
``` 
But, then as was mentioned in the lecture, the client would have to add machinery to handle the error message, and that also presents the problem of what the client should do in response to an error message.

I don't really see why killing the client is necessary when the client tries to deallocate a frequency it doesn't own, but another approach would be for the server to kill the client:
 

```erlang
        {request, Pid , {deallocate, Freq}} -> 
            try deallocate(Frequencies, Freq) of
                NewFrequencies -> 
                    Pid ! {reply, ok},
                    loop(NewFrequencies, Supervisor)
            catch
                error:{badmatch, _Val} ->
                    exit(Pid, not_allocated),
                    io:format("client (~w) deallocated bad frequency: ~w...killed client~n", [Pid, Freq]),
                    loop(Frequencies, Supervisor)
            end;
 ```
