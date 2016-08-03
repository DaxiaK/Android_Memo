# Http upload

## sample

This Class HttpURLConnection to simulate multipart/form-data of http post.

The nuiltipart\/form-data was defined : [http://www.ietf.org/rfc/rfc1867.txt](http://www.ietf.org/rfc/rfc1867.txt)

```
String lineEnd = "\r\n";
//Boundary MUST NOT be same as file content
String boundary = generateBoundary();
String twoHyphens = "--";
DataOutputStream dos = null;
int bytesRead, bytesAvailable, bufferSize;
byte[] buffer;
int maxBufferSize = 1 * 1024 * 1024;
Activity activity = weakRefActivity.get();
if (activity != null) {
 try {
 // open a URL connection to the Servlet
 URL url = new URL(mUrl + uploadURL);
 //If you will add other input way , please add it in here.
 ParcelFileDescriptor parcelFileDescriptor = activity.getContentResolver().openFileDescriptor(uri, "r");
 FileDescriptor fileDescriptor = parcelFileDescriptor.getFileDescriptor();
 FileInputStream fileInputStream = new FileInputStream(fileDescriptor);
 // Open a HTTP connection to the URL
 conn = (HttpURLConnection) url.openConnection();
 conn.setReadTimeout(READ_TIMEOUT);
 conn.setConnectTimeout(CONNECT_TIMEOUT);
 // Allow Inputs
 conn.setDoInput(true);
 // Allow Outputs
 conn.setDoOutput(true);
 // Don't use a Cached Copy
 conn.setUseCaches(false);
 conn.setRequestMethod("POST");
 // disables Keep Alive
 conn.setRequestProperty("connection", "close");
 conn.setRequestProperty("ENCTYPE", "multipart/form-data");
 conn.setRequestProperty("Content-Type", "multipart/form-data;boundary=" + boundary);
 conn.setRequestProperty("Accept-Charset", CHARSET);
 //Use Chunked because we didn't know the file size!
 conn.setChunkedStreamingMode(0);
 dos = new DataOutputStream(conn.getOutputStream()); dos.writeBytes(twoHyphens + boundary + lineEnd);
 String content = "Content-Disposition: form-data; name=\"File\"; filename=\"" + fileName + "\"" + lineEnd;
 dos.write(content.getBytes(Charset.forName("utf-8")));
 dos.writeBytes(lineEnd);
 // create a buffer of maximum size
 bytesAvailable = fileInputStream.available();
 bufferSize = Math.min(bytesAvailable, maxBufferSize);
 buffer = new byte[bufferSize];
 // read file and write it into form...
 while (fileInputStream.read(buffer, 0, bufferSize) >0) {
 dos.write(buffer, 0, bufferSize);
 bytesAvailable = fileInputStream.available();
 bufferSize = Math.min(bytesAvailable, maxBufferSize);
 }
 // send multipart form data necesssary after file data...
 dos.writeBytes(lineEnd);
 //--boundary--\r\n
 dos.writeBytes(twoHyphens + boundary + twoHyphens + lineEnd);
 getRequest();
 //Notice the close sequence MUST be correct.
 //if u close inputstream befor outputstream,that is ok.
 //close the streams //
 dos.flush();
 dos.close();
 //clos the input stram
 fileInputStream.close();
 conn.disconnect();
 } catch (MalformedURLException ex) {
 Log.e(TAG, ex.toString());
 } catch (IOException e) {
 Log.e(TAG, e.toString());
 } catch (Exception e) {
 Log.e(TAG, e.toString());
 }
}

```

bondary

```
//The pool of ASCII chars to be used for generating a multipart boundary.
private final static char[] MULTIPART_CHARS_POOL = "-_1234567890abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ" .toCharArray();

/** * This method create random boundary , and it follow httpclient . * * @return Boundar */
protected String generateBoundary() {
 StringBuilder buffer = new StringBuilder();
 SecureRandom rand = new SecureRandom();
 int count = rand.nextInt(11) + 30; // a random size from 30 to 40
 for (int i = 0; i < count; i++) {
 buffer.append(MULTIPART_CHARS_POOL[rand.nextInt(MULTIPART_CHARS_POOL.length)]);
 }
 return buffer.toString();
}


```

