<!-- fetched 2026-10-13 from https://docs.digitalocean.com/products/spaces/reference/s3-compatibility/ by sweep-shape-compat.py; text extracted from the provider's own page -->

Spaces S3 Compatibility | DigitalOcean Documentation 
 Docs 
 Platform 
 Products 
 Reference 
 Support 
 Sign Up 
 Product Home 
 Spaces Object Storage 
 Getting Started 
 Quickstart 
 How-Tos 
 Create Spaces Bucket 
 Destroy Spaces Bucket 
 Manage Access 
 Migrate Buckets 
 Transfer Buckets Between Regions 
 Manage Files 
 Organize Files in Folders 
 Share Links to Files 
 Set File Listing Permissions 
 Set File Metadata 
 Enable the CDN 
 Manage the CDN Cache 
 Configure Access Logs 
 Configure Versioning 
 Configure Bucket Policies 
 Configure Lifecycle Rules 
 Configure CORS 
 Reference 
 API Reference 
 Reference 
 MCP Reference 
 S3 Compatibility 
 s3cmd 2.x Setup 
 s3cmd 2.x Usage 
 AWS S3 SDKs 
 Terraform Remote Backend 
 Transmit 4 
 Transmit 5 
 Cyberduck 6.2.x 
 Concepts 
 Performance Best Practices 
 Details 
 Features 
 Pricing 
 Availability 
 Limits 
 Reference 
 S3 Compatibility 
 Give Feedback 
 For AI agents: The documentation index is at https://docs.digitalocean.com/llms.txt . Markdown versions of pages use the same URL with index.html.md in place of the HTML page (for example, append index.html.md to the directory path instead of opening the HTML document).
 Spaces S3 Compatibility
 Last verified 22 Jun 2026
 Spaces Object Storage is an S3-compatible service for storing and serving large amounts of data. The built-in Spaces CDN minimizes page load times, improves performance, and reduces bandwidth and infrastructure costs.
 Copy page as Markdown View page as Markdown 
 DigitalOcean Spaces provides an S3-compatible API with partial support for Amazon S3 features.
 Feature Compatibility 
 The following table lists supported S3 features and restrictions:
 Feature | Notes | 
 Object Copy | Supported with CopyObject . Cross-region and cross-cluster copies are not supported. | 
 Multipart Uploads | Supported for large objects. UploadPartCopy is not supported across regions or clusters. | 
 Presigned URLs | Supported with both Signature Version 2 and Version 4. | 
 Bucket Policies | Supported only through the API (not configurable in the control panel). | 
 Bucket Versioning | Supported only through the API (not configurable in the control panel). | 
 Bucket Lifecycle | Supported for time-based expiration and removing incomplete multipart uploads. Tag-based lifecycle rules are not supported. | 
 Bucket Access Logging | Supported only through the API. The source and destination buckets must be different. | 
 Object Encryption | Supported with SSE-C (server-side encryption with customer-provided keys). Bucket-level encryption settings are not supported. | 
 Bucket Websites | Supported at https://<your-bucket-name>.<your-region>-static.digitaloceanspaces.com . CDN is not supported for bucket websites. | 
 List Objects | Both ListObjects (legacy) and ListObjectsV2 are supported. Use ListObjectsV2 for new applications. | 
 Preflight Requests | OPTIONS preflight requests require no permissions, but the bucket policy or ACL must still allow the subsequent request. | 
 Unsupported API Calls | Unsupported S3 operations return a standard NotImplemented error. | 
 Authentication | Spaces supports AWS Signature Version 4 (recommended) and Signature Version 2 for legacy clients. | 
 For more information on product-wide usage limits, see Spaces Limits .
 Access Control Lists (ACLs) 
 Spaces supports a limited set of canned ACLs for buckets and objects. Only two ACLs are available:
 private : Grants FULL_CONTROL to the bucket or object owner. Blocks public access.
 public-read : Grants FULL_CONTROL to the owner and allows unauthenticated read access to anyone on the internet.
 You can apply an ACL in two ways:
 Send a PUT ?acl request with an XML AccessControlPolicy .
 Use the x-amz-acl header with a canned ACL (simpler and recommended).
 Private ACL Example 
 To restrict access so only the bucket owner has full control, set the ACL to private. You can apply this setting either by adding the x-amz-acl header or by using an XML access control policy:
 x-amz-acl Header curl -X PUT "https://<your-example-space>.<your-region>.digitaloceanspaces.com/<your-example>.txt" \ -H "Authorization: <your-spaces-access-key>:<your-signature>" \ -H "Date: ..." \ -H "x-amz-acl: private" \ --upload-file ./<your-example>.txt 
 XML Policy <AccessControlPolicy xmlns= "http://s3.amazonaws.com/doc/2006-03-01/" > <Owner><ID> 6174283 </ID></Owner> <AccessControlList> <Grant> <Grantee xmlns:xsi= "http://www.w3.org/2001/XMLSchema-instance" xsi:type= "CanonicalUser" > <ID> 6174283 </ID> </Grantee> <Permission> FULL_CONTROL </Permission> </Grant> </AccessControlList> </AccessControlPolicy> 
 Public Read ACL Example 
 To make objects readable by anyone on the internet while still retaining full control as the owner, set the ACL to public-read. You can configure this either with the x-amz-acl header or with an XML access control policy:
 x-amz-acl Header curl -X PUT "https://<your-example-space>.<your-region>.digitaloceanspaces.com/<your-example>.txt" \ -H "Authorization: <your-spaces-access-key>:<your-signature>" \ -H "Date: ..." \ -H "x-amz-acl: public-read" \ --upload-file ./<your-example>.txt 
 XML Policy <AccessControlPolicy xmlns= "http://s3.amazonaws.com/doc/2006-03-01/" > <Owner><ID> 6174283 </ID><DisplayName> 6174283 </DisplayName></Owner> <AccessControlList> <Grant> <Grantee xmlns:xsi= "http://www.w3.org/2001/XMLSchema-instance" xsi:type= "Group" > <URI> http://acs.amazonaws.com/groups/global/AllUsers </URI> </Grantee> <Permission> READ </Permission> </Grant> <Grant> <Grantee xmlns:xsi= "http://www.w3.org/2001/XMLSchema-instance" xsi:type= "CanonicalUser" > <ID> 6174283 </ID> </Grantee> <Permission> FULL_CONTROL </Permission> </Grant> </AccessControlList> </AccessControlPolicy> 
 In this article...
 Feature Compatibility 
 Access Control Lists (ACLs) Private ACL Example 
 Public Read ACL Example 
 Company
 About 
 Careers 
 Blog 
 Docs
 Docs Home 
 API Reference 
 CLI Reference 
 Release Notes 
 llms.txt 
 Trust Platform 
 Community
 Tutorials 
 Q&A 
 Write for DOnations 
 Currents Research 
 Legal 
 Code of Conduct 
 Support
 Support Center 
 Report Abuse 
 © 2026 DigitalOcean, LLC. All rights reserved 
 We can't find any results for your search.
 Try using different keywords or simplifying your search terms.
