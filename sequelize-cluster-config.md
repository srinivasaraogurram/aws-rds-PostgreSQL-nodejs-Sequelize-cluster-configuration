const { Sequelize } = require('sequelize');
 
const sequelize = new Sequelize({
  database: 'database',
  username: 'username',
  password: 'password',
  dialect: 'postgres',
  replication: {
    read: [
      { host: 'read-replica-1.cluster-xyz.us-east-1.rds.amazonaws.com', username: 'username', password: 'password' },
      { host: 'read-replica-2.cluster-xyz.us-east-1.rds.amazonaws.com', username: 'username', password: 'password' }
    ],
    write: { host: 'primary-instance.cluster-xyz.us-east-1.rds.amazonaws.com', username: 'username', password: 'password' }
  },
  pool: {
    max: 10,
    min: 0,
    acquire: 30000,
    idle: 10000
  }
});
