
To enhance the script to handle failover in an AWS Aurora PostgreSQL cluster where you have an active (east) and passive (west) region, you need to account for the following:

1. **Detecting Failover**: Aurora automatically promotes the passive region to active during a failover. You need to detect this change and update your Sequelize configuration accordingly.
2. **Cross-Region Access**: Ensure your application can access the database in both regions.
3. **Dynamic Configuration Update**: Update the Sequelize configuration dynamically when a failover occurs.

Here’s how you can enhance the script to handle failover:

---

### Enhanced Script

```javascript
const AWS = require('aws-sdk');
const { Sequelize } = require('sequelize');

// Configure AWS SDK for both regions
AWS.config.update({ region: 'us-east-1' }); // Replace with your active region
const rdsEast = new AWS.RDS();
const rdsWest = new AWS.RDS({ region: 'us-west-2' }); // Replace with your passive region

// Function to get the writer instance from a specific region
async function getWriterInstance(clusterIdentifier, rdsClient) {
    const params = {
        DBClusterIdentifier: clusterIdentifier,
    };

    try {
        const data = await rdsClient.describeDBClusters(params).promise();
        const cluster = data.DBClusters[0];
        const writerInstance = cluster.DBClusterMembers.find(member => member.IsClusterWriter === true).DBInstanceIdentifier;
        return writerInstance;
    } catch (error) {
        console.error('Error describing DB cluster:', error);
        throw error;
    }
}

// Function to initialize Sequelize with the correct writer and reader endpoints
function initializeSequelize(writerEndpoint, readerEndpoints) {
    return new Sequelize('your-database-name', 'your-username', 'your-password', {
        host: writerEndpoint,
        dialect: 'postgres',
        replication: {
            read: readerEndpoints.map(endpoint => ({
                host: endpoint,
                username: 'your-username',
                password: 'your-password',
            })),
            write: {
                host: writerEndpoint,
                username: 'your-username',
                password: 'your-password',
            },
        },
        pool: {
            max: 10,
            min: 0,
            idle: 10000,
        },
    });
}

// Function to check for failover and update Sequelize configuration
async function checkForFailoverAndUpdateSequelize(clusterIdentifier, sequelize) {
    try {
        // Check the active region (east) first
        let writerInstance = await getWriterInstance(clusterIdentifier, rdsEast);
        console.log('Current Writer Instance (East):', writerInstance);

        // If no writer instance is found in the active region, check the passive region (west)
        if (!writerInstance) {
            writerInstance = await getWriterInstance(clusterIdentifier, rdsWest);
            console.log('Current Writer Instance (West):', writerInstance);
        }

        // Update Sequelize configuration if the writer instance has changed
        if (writerInstance && sequelize.config.replication.write.host !== writerInstance) {
            console.log('Failover detected. Updating Sequelize configuration...');
            const readerEndpoints = [/* Add your reader endpoints here */];
            sequelize = initializeSequelize(writerInstance, readerEndpoints);
            console.log('Sequelize configuration updated.');
        }

        return sequelize;
    } catch (error) {
        console.error('Error during failover check:', error);
        throw error;
    }
}

// Example usage
(async () => {
    const clusterIdentifier = 'your-cluster-identifier';
    let sequelize = initializeSequelize('initial-writer-endpoint', ['reader-endpoint-1', 'reader-endpoint-2']);

    // Periodically check for failover and update Sequelize configuration
    setInterval(async () => {
        sequelize = await checkForFailoverAndUpdateSequelize(clusterIdentifier, sequelize);
    }, 60000); // Check every 60 seconds

    // Test the connection
    sequelize.authenticate()
        .then(() => {
            console.log('Connection has been established successfully.');
        })
        .catch(err => {
            console.error('Unable to connect to the database:', err);
        });
})();
```

---

### Key Enhancements

1. **Cross-Region RDS Clients**:
   - Two `AWS.RDS` clients are created, one for the active region (east) and one for the passive region (west).

2. **Failover Detection**:
   - The `getWriterInstance` function checks for the writer instance in the active region first. If no writer is found, it checks the passive region.

3. **Dynamic Sequelize Update**:
   - The `checkForFailoverAndUpdateSequelize` function compares the current writer instance with the one in the Sequelize configuration. If they differ, it updates the Sequelize object with the new writer and reader endpoints.

4. **Periodic Failover Check**:
   - A `setInterval` is used to periodically check for failover and update the Sequelize configuration.

5. **Reader Endpoints**:
   - Replace `['reader-endpoint-1', 'reader-endpoint-2']` with the actual reader endpoints for your cluster.

---

### Notes

- **Cross-Region Access**:
  - Ensure your application has network access to both regions (e.g., via VPC peering or public endpoints).
  - Update IAM roles and security groups to allow cross-region access.

- **Failover Handling**:
  - Aurora automatically handles failover, but your application needs to detect and adapt to the new writer instance.

- **Error Handling**:
  - Add robust error handling for network issues, authentication failures, etc.

- **Performance**:
  - Adjust the failover check interval (`setInterval`) based on your application's requirements.

This enhanced script ensures your application dynamically adapts to failover events in an AWS Aurora PostgreSQL cluster.
