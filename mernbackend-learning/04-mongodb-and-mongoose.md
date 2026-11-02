# 04. MongoDB and Mongoose

MongoDB stores BSON documents in collections. Mongoose adds application-level schemas, casting, validation, and model methods. Database constraints such as unique indexes remain important because concurrent requests can bypass a check performed only in application code.

## Schema and model

```js
import mongoose from 'mongoose';

const doctorSchema = new mongoose.Schema({
  name: { type: String, required: true, trim: true },
  city: { type: String, required: true, index: true },
  verified: { type: Boolean, default: false },
}, { timestamps: true });

export const Doctor = mongoose.model('Doctor', doctorSchema);
```

Use references when related records have their own lifecycle or can grow without bound. Embed small related values when they are usually read and updated with their parent. Choose based on access patterns and consistency needs.

## Queries and indexes

Select only fields needed by a response, paginate list queries, and create indexes for frequent filters and sorts. Indexes speed supported reads but consume storage and add work to writes. A unique index enforces uniqueness at the database level; catch duplicate-key errors and translate them into a safe client response.

Never pass arbitrary client objects directly into Mongo queries or updates. Build filters and allowed update fields explicitly to prevent unintended query operators or mass assignment.

## Practice

Model appointments with references to doctor and patient. Add an index matching a common query, such as appointments by doctor and date, and explain how you would check whether the query uses it.
