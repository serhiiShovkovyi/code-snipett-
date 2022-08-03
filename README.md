#Image Library component


#Issue:
CRM system for advertisements has 2 types of campaign template builders
One for creating campaign from blocks, the second for writing pure HTML
Second, which is HTML editor, should have an image selection for partnets’ saved images.

#Task:
Develop image library for HTML editor according to Client requirement

#Client requirement:
We should have a modal which opens from the right corner.
Modal contains 3 tabs, Upload image, Select Image from Library, Import Image by URL
Each tab has a file info box in the bottom (file info contains: File Size, File Name, create Date) and 2 buttons under the file info box.
First tab - The Upload image has a drop-down field on top
The second tab - Select image has a list of images with the search bar, images are loading by 20 each time the user scrolls to the bottom.
The third tab - Import Image, has a field for paste URLs.
Each loaded image automatically added to the Second tab
Once you select an image, you have the option to add it to an HTML editor.


Technical task:
According to the Organism design pattern, we should:
Create modal window component which follows such architecture

ImageLibraryModal →
	Segments (modal tabs)/
	UploadImageArea/
	ImageLibrary
	ImportImageFromLink/
	SelectedImageInfo/
ImageLibraryModal←






Components description:

ImageLibraryModal
	props: size(String), type(String), status(Boolean), title(String), accept-button-text(String), cancel-button-text(String)

methods:
closeModal - clears data and closes modal
acceptAction - adds selected image 
changeSegment - handles tab change
handleScroll - once scrolled down, toggles loadLibList
closeImgLib - once closed image library tab toggles clearLib
copyUrl - copies saved internal image url 
loadLibList - loads initial 20 images
clearLib - dumps loaded image list
loadMoreImages - adds 20 to params and load images according to gap (20-40, 40-60 .. etc)
selectImage - called once used clicks on image and load file data for SelectedImageInfo component
deleteCurrentImg - deletes selected image
imageUploaded - once file of image is loaded, creates a request to save it to the database and AWS cloud server
addImage - toggles by UploadImageArea and send file to the checkAndLoadImage
uploadFromLink - toggled by ImportImageFromLink and send blob to the checkAndLoadImage
checkAndLoadImage - validates file or blob, checks cors policy for ImportImageFromLink and sends ready form to the imageUploaded
toggleAddImageModal - oppens modal
