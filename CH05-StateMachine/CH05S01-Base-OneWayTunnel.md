# 基础组件：OneWayTunnel

OneWayTunnel 是使用 IOCore 网络子系统的 API 实现的一个完整的状态机。对于理解和学习如何使用iocore系统进行开发，这是一个很好的例子。

OneWayTunnel 实现了一个简单的数据传输功能，它从一个VC读取数据然后将数据写入另外一个VC。

## 定义

```
//////////////////////////////////////////////////////////////////////////////
//
//      OneWayTunnel
//
//////////////////////////////////////////////////////////////////////////////

#define TUNNEL_TILL_DONE INT64_MAX

#define ONE_WAY_TUNNEL_CLOSE_ALL NULL

typedef void (*Transform_fn)(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf);

/**
  A generic state machine that connects two virtual conections. A
  OneWayTunnel is a module that connects two virtual connections, a source
  vc and a target vc, and copies the data between source and target. Once
  the tunnel is started using the init() call, it handles all the events
  from the source and target and optionally calls a continuation back when
  its done. On success it calls back the continuation with VC_EVENT_EOS,
  and with VC_EVENT_ERROR on failure.

  If manipulate_fn is not NULL, then the tunnel acts as a filter,
  processing all data arriving from the source vc by the manipulate_fn
  function, before sending to the target vc. By default, the manipulate_fn
  is set to NULL, yielding the identity function. manipulate_fn takes
  a IOBuffer containing the data to be written into the target virtual
  connection which it may manipulate in any manner it sees fit.

*/
struct OneWayTunnel : public Continuation {
  //
  //  Public Interface
  //

  //  Copy nbytes from vcSource to vcTarget.  When done, call
  //  aCont back with either VC_EVENT_EOS (on success) or
  //  VC_EVENT_ERROR (on error)
  //

  // Use these to construct/destruct OneWayTunnel objects

  /**
    Allocates a OneWayTunnel object.

    @return new OneWayTunnel object.

  */
  static OneWayTunnel *OneWayTunnel_alloc();

  /** Deallocates a OneWayTunnel object. */
  static void OneWayTunnel_free(OneWayTunnel *);

  static void SetupTwoWayTunnel(OneWayTunnel *east, OneWayTunnel *west);
  OneWayTunnel();
  virtual ~OneWayTunnel();

  // Use One of the following init functions to start the tunnel.
  /**
    This init function sets up the read (calls do_io_read) and the write
    (calls do_io_write).

    @param vcSource source VConnection. A do_io_read should not have
      been called on the vcSource. The tunnel calls do_io_read on this VC.
    @param vcTarget target VConnection. A do_io_write should not have
      been called on the vcTarget. The tunnel calls do_io_write on this VC.
    @param aCont continuation to call back when the tunnel finishes. If
      not specified, the tunnel deallocates itself without calling back
      anybody. Otherwise, its the callee's responsibility to deallocate
      the tunnel with OneWayTunnel_free.
    @param size_estimate size of the MIOBuffer to create for
      reading/writing to/from the VC's.
    @param aMutex lock that this tunnel will run under. If aCont is
      specified, the Continuation's lock is used instead of aMutex.
    @param nbytes number of bytes to transfer.
    @param asingle_buffer whether the same buffer should be used to read
      from vcSource and write to vcTarget. This should be set to true in
      most cases, unless the data needs be transformed.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target if true, the tunnel closes vcTarget at the
      end. If aCont is not specified, this should be set to true.
    @param manipulate_fn if specified, the tunnel calls this function
      with the input and the output buffer, whenever it gets new data
      in the input buffer. This function can transform the data in the
      input buffer
    @param water_mark watermark for the MIOBuffer used for reading.

  */
  void init(VConnection *vcSource, VConnection *vcTarget, Continuation *aCont = NULL, int size_estimate = 0, // 0 = best guess
            ProxyMutex *aMutex = NULL, int64_t nbytes = TUNNEL_TILL_DONE, bool asingle_buffer = true, bool aclose_source = true,
            bool aclose_target = true, Transform_fn manipulate_fn = NULL, int water_mark = 0);

  /**
    This init function sets up only the write side. It assumes that the
    read VConnection has already been setup.

    @param vcSource source VConnection. Prior to calling this
      init function, a do_io_read should have been called on this
      VConnection. The tunnel uses the same MIOBuffer and frees
      that buffer when the transfer is done (either successful or
      unsuccessful).
    @param vcTarget target VConnection. A do_io_write should not have
      been called on the vcTarget. The tunnel calls do_io_write on
      this VC.
    @param aCont The Continuation to call back when the tunnel
      finishes. If not specified, the tunnel deallocates itself without
      calling back anybody.
    @param SourceVio VIO of the vcSource.
    @param reader IOBufferReader that reads from the vcSource. This
      reader is provided to vcTarget.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target if true, the tunnel closes vcTarget at the
      end. If aCont is not specified, this should be set to true.
  */
  void init(VConnection *vcSource, VConnection *vcTarget, Continuation *aCont, VIO *SourceVio, IOBufferReader *reader,
            bool aclose_source = true, bool aclose_target = true);

  /**
    Use this init function if both the read and the write sides have
    already been setup. The tunnel assumes that the read VC and the
    write VC are using the same buffer and frees that buffer
    when the transfer is done (either successful or unsuccessful)
    @param aCont The Continuation to call back when the tunnel finishes. If
    not specified, the tunnel deallocates itself without calling back
    anybody.

    @param SourceVio read VIO of the Source VC.
    @param TargetVio write VIO of the Target VC.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target if true, the tunnel closes vcTarget at the
      end. If aCont is not specified, this should be set to true.

    */
  void init(Continuation *aCont, VIO *SourceVio, VIO *TargetVio, bool aclose_source = true, bool aclose_target = true);

  //
  // Private
  //
  OneWayTunnel(Continuation *aCont, Transform_fn manipulate_fn = NULL, bool aclose_source = false, bool aclose_target = false);

  int startEvent(int event, void *data);

  virtual void transform(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf);

  /** Result is -1 for any error. */
  void close_source_vio(int result);

  virtual void close_target_vio(int result, VIO *vio = ONE_WAY_TUNNEL_CLOSE_ALL);

  void connection_closed(int result);

  virtual void reenable_all();

  bool last_connection();

  VIO *vioSource;
  VIO *vioTarget;
  Continuation *cont;
  Transform_fn manipulate_fn;
  int n_connections;
  int lerrno;

  bool single_buffer;
  bool close_source;
  bool close_target;
  bool tunnel_till_done;

  /** Non-NULL when this is one side of a two way tunnel. */
  OneWayTunnel *tunnel_peer;
  bool free_vcs;

private:
  OneWayTunnel(const OneWayTunnel &);
  OneWayTunnel &operator=(const OneWayTunnel &);
};
```

## 方法


## 参考资料

- [I_OneWayTunnel.h](http://github.com/apache/trafficserver/tree/master/iocore/utils/I_OneWayTunnel.h)

