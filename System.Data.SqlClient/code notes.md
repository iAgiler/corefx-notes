
```
namespace System.Data.SqlClient
{
    public sealed partial class SqlConnection : DbConnection, ICloneable
    {
        // ...

        public override Task OpenAsync(CancellationToken cancellationToken)
        {
            // ...
            System.Transactions.Transaction transaction = ADP.GetCurrentTransaction();
            /*
            internal static Transaction GetCurrentTransaction()
            {
                return Transaction.Current;
            }
            */
        }
        
        private bool TryOpen(TaskCompletionSource<DbConnectionInternal> retry)
        {
            // ...
            if (!tdsInnerConnection.ConnectionOptions.Pooling)
            {
                // For non-pooled connections, we need to make sure that the finalizer does actually run to avoid leaking SNI handles
                GC.ReRegisterForFinalize(this);
            }
            // ...
        }
        // ...

        private void DisposeMe(bool disposing)
        {
            _credential = null;
            _accessToken = null;

            if (!disposing)
            {
                // For non-pooled connections we need to make sure that if the SqlConnection was not closed,
                // then we release the GCHandle on the stateObject to allow it to be GCed
                // For pooled connections, we will rely on the pool reclaiming the connection
                var innerConnection = (InnerConnection as SqlInternalConnectionTds);
                if ((innerConnection != null) && (!innerConnection.ConnectionOptions.Pooling))
                {
                    var parser = innerConnection.Parser;
                    if ((parser != null) && (parser._physicalStateObj != null))
                    {
                        parser._physicalStateObj.DecrementPendingCallbacks(release: false);
                    }
                }
            }
        }

        // ...
    }
}
```

```
namespace System.Data.ProviderBase
{
    internal abstract partial class DbConnectionFactory
    {        
        internal bool TryGetConnection(DbConnection owningConnection, TaskCompletionSource<DbConnectionInternal> retry, DbConnectionOptions userOptions, DbConnectionInternal oldConnection, out DbConnectionInternal connection)
        {
            // ...
            DbConnectionPool connectionPool;
            // ...
            do
            {
                connectionPool = GetConnectionPool(owningConnection, poolGroup);
                if (null == connectionPool)
                {
                    // If GetConnectionPool returns null, we can be certain that
                    // this connection should not be pooled via DbConnectionPool
                    // or have a disabled pool entry.
                    // ...
                }
                else
                {
                     if (((SqlClient.SqlConnection)owningConnection).ForceNewConnection)
                    {
                        // ...
                        connection = connectionPool.ReplaceConnection(owningConnection, userOptions, oldConnection);
                    }
                    else
                    {
                        if (!connectionPool.TryGetConnection(owningConnection, retry, userOptions, out connection))
                        {
                            return false;
                        }
                    }

                    if (connection == null)
                    {
                        // connection creation failed on semaphore waiting or if max pool reached
                        if (connectionPool.IsRunning)
                        {
                            // If GetConnection failed while the pool is running, the pool timeout occurred.
                            throw ADP.PooledOpenTimeout();
                        }
                        else
                        {
                            // We've hit the race condition, where the pool was shut down after we got it from the group.
                            // Yield time slice to allow shut down activities to complete and a new, running pool to be instantiated
                            //  before retrying.
                            Threading.Thread.Sleep(timeBetweenRetriesMilliseconds);
                            timeBetweenRetriesMilliseconds *= 2; // double the wait time for next iteration
                        }
                    }
                }
            } while (connection == null && retriesLeft-- > 0);

            if (connection == null)
            {
                // exhausted all retries or timed out - give up
                throw ADP.PooledOpenTimeout();
            }

            return true;
        }
    }
}
```

```
namespace System.Data.SqlClient
{
    internal sealed partial class SqlConnectionString : DbConnectionOptions
    {
        // instances of this class are intended to be immutable, i.e readonly
        // used by pooling classes so it is much easier to verify correctness
        // when not worried about the class being modified during execution

        internal static partial class DEFAULT
        {
            // ...
            internal const bool Enlist = true;
            // ...
            internal const int Max_Pool_Size = 100;
            internal const int Min_Pool_Size = 0;
            // ...
            internal const bool Pooling = true;
            // ...
        }
    }
}
```