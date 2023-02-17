# Examples of code with express 

## Upload files to firebase with express and busboy

Install the dependencies
```
npm i busboy firebase-functions express
```
Import the dependencies
```node
const Busboy = require('busboy');
const { Storage } = require('@google-cloud/storage');
```
Create instances of the storage and the bucket
```node
const storage = new Storage({ keyFilename: "permissions.json" }); 
const bucket = storage.bucket('buckename');
```
Create a function to get the form data
```node
const parseForm = async req => {
    return new Promise((resolve, reject) => {
        const busboy = Busboy({ headers: req.headers, limits: { fileSize: 20 * 1024 * 1024 } });
        const files = [] // create an empty array to hold the processed files
        const buffers = {} // create an empty object to contain the buffers
        const fields = [] // fields of the req.body
        busboy.on('file', async (name, file, info) => {
            field = info.filename;
            buffers[name] = [] // add a new key to the buffers object
            
            file.on('data', data => {
                buffers[name].push(data)
            })
            file.on('close', () => {
                files.push({
                    fileBuffer: Buffer.concat(buffers[name]),
                    fileType: info.mimeType,
                    fileName: info.filename,
                    fileEnc: info.encoding,
                });

            })
        })
        busboy.on('field', (fieldname, val) => {
            fields[fieldname] = val;
        })
        busboy.on('error', err => {
            reject(err)
        })
        busboy.on('close', () => {
            resolve({ files, fields })
        })
        busboy.end(req.rawBody);
        req.pipe(busboy) // pipe the request to the form handler
    })
}
```
Create a function to upload the files to firebase
```node
const uploadFile = async (file, folder) => new Promise ((resolve, reject) => {
    console.log('LOG: subiendo archivo...');
    const { fileBuffer, ...fileParams } = file;
    const fileName = fileParams.fileName.replaceAll(' ', '');;
    let blob = bucket.file(fileName);


    const blobStream = blob.createWriteStream({
        resumable: false,
        contentType: fileParams.fileType,
        validation: "md5"
    });

    console.log('LOG: creando el blob stream ...')
    blobStream.on('finish', async () => {
        await bucket.file(fileName).move(`${folder}${fileName}`);
        await bucket.file(`${folder}${fileName}`).makePublic();
        const publicUrl = `https://storage.googleapis.com/${bucket.name}/${folder}${fileName}`;
        console.log('LOG: Archivo subido ... ', publicUrl);
        resolve(publicUrl)
    });

    blobStream.on('error', error => {
            console.log('LOG: Error al subir el archivo ... ', error);
            reject(`Unable to upload file, something went wrong - `, error)
            
    });
    
    blobStream.end(fileBuffer);
});
```
Execute both functions

```node
const { files, fields } = await globalHelper.parseForm(req);
// Example of getting a field
let example = fields["example"];
let folder = "/example";
await globalHelper.uploadFile(files[0], folder).then((result) => {
   fileUrls.push(result); //urls of the files uploaded
});
```
