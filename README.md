perc is an Erlang interface for controlling Unix processes.

## COMPILING

    make


## EXPORTS

### perc

    kill(Pid, Signal) -> ok | {error, posix()}
    
        Types   Pid = integer()
                Signal = integer()

        Send a signal to a Unix process.

        Process IDs and signals are signed 32-bit integers.

        Sending a signal to PID 0 will send the signal to the Erlang VM.

    getpriority(Which, Who) -> {ok, integer()} | {error, posix()}

        Types   Which = integer()
                Who = integer()

        See getpriority(2).

        Get the priority (the "nice" value) of processes by pid, process
        group or user.

    setpriority(Which, Who, Prio) -> ok | {error, posix()}

        Types   Which = integer()
                Who = integer()
                Prio = integer()

        See setpriority(2).

        Set the priority (the "nice" value) of processes by pid, process
        group or user.

    renice(Type, Prio) -> {ok, NewPrio} | {error, posix()}

        Types   Type = {Which, Who}
                Which = pid | pgrp | user
                Who = integer()
                Prio = integer() | string()
                NewPrio = integer()

        Convenience wrapper around getpriority/2 and setpriority/3,
        similar to renice(1).

        WARNING: renice/2 makes successive calls to getpriority/2 and
        setpriority/3. Since this sequence is not atomic, the priority
        may change between calls or the process may have been terminated.

        Sets the priority of a process or processes by pid, pgroup or
        user and returns the new priority. The new priority may be an
        integer or a list containing a relative priority, indicated by
        using a "+" or "-". For example, using "+10" will increase the
        niceness of the process by 10.

    prlimit(Pid, Resource, NewLimit, OldLimit) -> {ok, NewLimit1, OldLimit1} | {error, Error}

        Types   Pid = integer()
                Resource = integer()
                    | rlimit_cpu
                    | rlimit_fsize
                    | rlimit_data
                    | rlimit_stack
                    | rlimit_core
                    | rlimit_rss
                    | rlimit_nproc
                    | rlimit_nofile
                    | rlimit_ofile
                    | rlimit_memlock
                    | rlimit_as
                    | rlimit_locks
                    | rlimit_sigpending
                    | rlimit_msgqueue
                    | rlimit_nice
                    | rlimit_rtprio
                    | rlimit_rttime
                    | rlimit_nlimits
                    | rlim_infinity
                NewLimit = NewLimit1 = <<>> | binary()
                OldLimit = OldLimit1 = <<>> | binary()
                Error = unsupported | posix()

        Linux only: on other platforms, {error, unsupported} will be
        returned to the caller.

        Set or retrieve process limits for a process. Passing in an
        empty binary for NewLimit or OldLimit indicates the caller is
        not interested in these values.

        The binary size of NewLimit/OldLimit must otherwise match the size
        of a struct rlimit for the platform. struct rlimit is usually
        composed of two 8 byte values in native format. To retrieve the
        current settings, pass in a zeroed 16 byte value.

    getrlimit(Resource) -> {ok, Limit} | {error, Error}

        Types Resource = integer() | rlimit()
              Error = posix()

        Get process limits for beam. See prlimit/4 for a list of the
        resource atoms.

        The value returned is a struct rlimit:

            1> {ok, <<Soft:8/native-unsigned-integer-unit:8, Hard:8/native-unsigned-integer-unit:8>>} = perc:getrlimit(rlimit_nofile).
            2> Soft.
            1024
            3> Hard.
            4096

    setrlimit(Resource, Limit) -> {ok, Limit} | {error, Error}

        Types Resource = integer() | rlimit()
              Error = posix()
              Limit = binary()

        Set process limits for beam. See prlimit/4 for a list of the
        resource atoms.

    umask() -> CurMask
    umask(Mask) -> OldMask

        Types   Mask = integer() | list()
                CurMask = OldMask = integer()

        Sets the file creation mask for beam. The mask may be either
        an integer or a list representing an octal number, e.g., either
        8#022 or "022".

        The old mask value is returned. To retrieve the current umask,
        use umask/0.

        WARNING: querying the umask is destructive: it is done by
        setting the umask to 0, then re-setting it back to the original
        umask. Between successive calls to umask/1, the process may be
        swapped out. If another process queries the umask, the beam Unix
        process may end up in an inconsistent state.

        To avoid this race condition, umask/0 sets a lock by using
        process registration. This may have some unintended side effects:

        1. If the atom is registered by something other than perc,
           umask/0 will spin forever.

        2. If, between successive calls to umask/1, something else
           (another NIF or driver) changes the umask, umask/0 will crash,
           releasing the lock.

        Neither of the above is likely to happyen but at some point,
        the umask/0 code may be moved into C to prevent these problems.


### perc_signal

_WARNING: support for signalfd(2) is experimental only and is disabled by default._

To enable, add `-DHAVE_SIGNALFD` to the rebar.config tuple for your
architecture. Some of the signals may be caught by beam. signalfd is
likely to work only with a single threaded beam. The interaction between
signals and POSIX threads is mysterious.

        start(Signals) -> {ok, pid()} | {error, enomem}

            Types   Signals = [ Signal ]
                    Signal = integer() | Names
                    Names = sighup
                        | sigint
                        | sigquit
                        | sigill
                        | sigtrap
                        | sigabrt
                        | sigbus
                        | sigfpe
                        | sigkill
                        | sigusr1
                        | sigsegv
                        | sigusr2
                        | sigpipe
                        | sigalrm
                        | sigterm
                        | sigstkflt
                        | sigchld
                        | sigcont
                        | sigstop
                        | sigtstp
                        | sigttin
                        | sigttou
                        | sigurg
                        | sigxcpu
                        | sigxfsz
                        | sigvtalrm
                        | sigprof
                        | sigwinch
                        | sigio
                        | sigpwr
                        | sigsys
                        | sigrtmin
                        | sigrtmax

        Linux only: on other platforms, {error, unsupported} will be
        returned to the caller.

        Receive notification when the beam process receives a signal:

            {signal, pid(), #signalfd_siginfo{}}

        The tuple contains the PID of the gen_server and information
        about the signal. See signalfd(2) for details about the returned
        data. For example, to match the signal number:

            -include_lib("perc/include/perc_signal.hrl").

            start() ->
                {ok, Ref} = perc_signal:start([sighup]),

                receive
                    {signal, Ref, #signalfd_siginfo{
                        ssi_signo = 1,
                        ssi_pid = Pid,
                        ssi_uid = UID,
                        }} ->
                        error_logger:info_report([
                            {sending_pid, Pid},
                            {sending_uid, UID}
                            ])
                end.

    stop(Ref) -> ok

        Stop the gen_server and close the signalfd file descriptor.


## EXAMPLES

### kill(2)

    $ sleep 1000000 &
    [1] 2947

    $ erl -pa ebin

    1> perc:kill(2947, 9).
    ok

    2> perc:kill(1, 1).
    {error,eperm}

    3> perc:kill(31337, 1).
    {error,esrch}

### prlimit(2)

    % Get the current value for rlimit_cpu
    1> perc:prlimit(0, rlimit_cpu, <<>>, <<0:128>>).
    {ok,<<>>,<<"ÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿ">>}

    % Set RLIMIT_NOFILE (7) on the Erlang VM. Sets the current and max file descriptors
    % to 8.
    1> perc:prlimit(0, 7, <<8:8/native-unsigned-integer-unit:8, 8:8/native-unsigned-integer-unit:8>>, <<>>).
    {ok,<<8,0,0,0,0,0,0,0,8,0,0,0,0,0,0,0>>,<<>>}

    2> file:open("/etc/passwd", [read]).
    {ok,<0.39.0>}

    3> file:open("/etc/passwd", [read]).
    {error,emfile}

### setpriority(2)/getpriority(2)

    % PRIO_PROCESS = 0
    1> perc:getpriority(0, 0).
    {ok, 0}

    2> perc:setpriority(0, 0, 10).
    ok

    3> perc:getpriority(0, 0).
    {ok, 10}

    4> perc:renice({pid, 0}, "-5").
    {ok, 5}

## TODO

* process lookup

* atom rlimit\_\* constants only supported for 64-bit values
