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
 
 I didn't really see why killing the client is necessary when the client tries to deallocate a frequency it doesn't own, but another 
