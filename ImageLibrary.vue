<template>
    <in-modals
        type="dynamic"
        size="small"
        :status="status"
        :title="trans('newsletter.add-image')"
        :accept-button-text="trans('products.add')"
        :cancel-button-text="trans('newsletter.cancel')"
        @closeModal="closeModal"
        @acceptButton="acceptAction">
        <div
            :class="{'image-info-offset': isShowImageInfo}"
            class="h-1 d-g grid-rows-100">
            <in-segments
                class="w-1 mb-4"
                align="horizontal"
                :disable="uploadingStatus"
                :segment-list="segmentList"
                :selected="currentSegment"
                @click="changeSegment">
            </in-segments>
            <upload-image-area
                v-if="currentSegment === imageLibSegments.UPLOAD_IMAGE"
                ref="uploadArea"
                :img-name="currentImage.title"
                :image-src="currentImage.cloudUrl"
                :uploading-status="uploadingStatus"
                :uploaded-file-invalid="uploadedFileInvalid"
                :image-load-error="imageLoadError"
                :error-message="imageUploadError"
                @uploadedFile="imageUploaded"
                @removeFile="deleteCurrentImg(currentImage.id)">
            </upload-image-area>
            <image-library
                v-if="currentSegment === imageLibSegments.IMAGE_LIBRARY"
                :image-list="imageLibList"
                :image-library-loading-status="imageLibraryLoadingStatus"
                @selectImage="selectImage"
                @searchInImages="input => loadLibList(input)"
                @deleteImage="(deletedImageId, isWhileSearch) =>
                    deleteCurrentImg(deletedImageId, true, isWhileSearch)"
                @handleScroll="handleScroll">
            </image-library>
            <add-image-from-link
                v-if="currentSegment === imageLibSegments.FROM_LINK"
                :image-preview-src="imageFromLinkPreviewUrl"
                :is-url-invalid="isUrlInvalid"
                :cors-error="corsError"
                :error-message="imageUploadError"
                @checkAndLoadImage="checkAndLoadImage">
            </add-image-from-link>
            <selected-image-info
                v-if="isShowImageInfo"
                :file-name="currentImage.title"
                :size="currentImage.size"
                :image-url="currentImage.cloudUrl"
                :copy-button-disabled="copyButtonDisabled"
                @copyURL="copyUrl">
            </selected-image-info>
        </div>
    </in-modals>
</template>

<script>
    import {
        checkFileType,
        copyTextToClipBoard,
        isEmpty,
        replaceString,
        isEmptyString
    } from '@/helpers';
    import {
        EndPoints,
        Limits,
        ImageLibSegments
    } from '@/_enums/NewsletterEnums';
    import { InModals, InSegments } from '@useinsider/design-system-vue';
    import UploadImageArea from 'Organisms/newsletter/add-image/UploadImageArea.vue';
    import ImageLibrary from 'Organisms/newsletter/add-image/ImageLibrary.vue';
    import AddImageFromLink from 'Organisms/newsletter/add-image/AddImageFromLink.vue';
    import SelectedImageInfo from 'Organisms/newsletter/add-image/SelectedImageInfo.vue';

    // eslint-disable-next-line no-useless-escape
    const VALID_URL_REGEXP = /[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_\+.~#?&//=]*)?/gi;
    const MAX_FILE_SIZE = 3145728;

    export default {
        components: {
            InModals,
            UploadImageArea,
            ImageLibrary,
            AddImageFromLink,
            InSegments,
            SelectedImageInfo
        },

        props: {
            /**
             * @property {boolean} status
             */
            status: {
                type: Boolean,
                default: false
            }
        },

        data () {
            return {
                segmentList: [
                    { text: this.trans('newsletter.upload-image'), value: ImageLibSegments.UPLOAD_IMAGE },
                    { text: this.trans('newsletter.image-library'), value: ImageLibSegments.IMAGE_LIBRARY },
                    { text: this.trans('newsletter.from-link'), value: ImageLibSegments.FROM_LINK }
                ],
                imagePreviewSrc: '',
                imageLibSegments: ImageLibSegments,
                copyButtonDisabled: false,
                imageLibraryModalStatus: false,
                imgLibraryAcceptButtonDisabled: true,
                acceptedImageTypes: [
                    'image/png',
                    'image/jpeg',
                    'image/jpg',
                    'image/gif',
                    'image/webp'
                ],
                currentImage: {},
                imageLibList: [],
                imageLibraryLoadingStatus: false,
                isUrlInvalid: false,
                loadedCount: Limits.LOAD_IMAGES_COUNT,
                uploadingStatus: false,
                uploadedFileInvalid: false,
                imageFromLinkPreviewUrl: '',
                uploadedImageBlob: new Blob(),
                imageLoadError: false,
                currentSegment: ImageLibSegments.UPLOAD_IMAGE,
                uploadImg: ImageLibSegments.UPLOAD_IMAGE,
                imgLibrary: ImageLibSegments.IMAGE_LIBRARY,
                fromLink: ImageLibSegments.FROM_LINK,
                corsError: false,
                disableAccept: false,
                imageUploadError: ''
            };
        },

        computed: {
            /**
             * @return {boolean}
             */
            isShowImageInfo () {
                return this.currentImage.title && this.currentImage.cloudUrl;
            }
        },

        watch: {
            /**
             * @return {void}
             */
            status () {
                this.disableAccept = false;
                this.copyButtonDisabled = false;

                if (this.currentSegment === ImageLibSegments.IMAGE_LIBRARY) {
                    this.loadLibList();
                }
            }
        },

        methods: {
            /**
             * @return {void}
             */
            async closeModal () {
                this.imageFromLinkPreviewUrl = '';
                this.isUrlInvalid = false;
                this.$emit('closeModal');

                if (this.currentSegment === ImageLibSegments.UPLOAD_IMAGE) {
                    await this.deleteCurrentImg(this.currentImage.id);
                }

                this.currentImage = {};
                this.imageUploadError = '';
                this.corsError = false;
            },

            /**
             * @return {void}
             */
            acceptAction () {
                if (!this.isShowImageInfo || this.disableAccept) {
                    this.$message.show(this.trans('newsletter.enter-valid-to-continue'), 'warning');

                    return;
                }

                this.disableAccept = true;

                if (this.currentSegment === ImageLibSegments.FROM_LINK) {
                    this.uploadFromLink(this.uploadedImageBlob);

                    return;
                }

                this.addImage(this.currentImage);
            },

            /**
             * @param {string} newSegment
             * @return {void}
             */
            async changeSegment (newSegment) {
                const prevSegment = this.currentSegment;

                this.currentSegment = newSegment;

                if (prevSegment === this.uploadImg && !isEmpty(this.currentImage)) {
                    await this.deleteCurrentImg(this.currentImage.id, true);
                }

                this.clearLib();

                if (newSegment === this.imgLibrary) {
                    this.loadLibList();
                }
            },

            /**
             * @param {object} el
             * @return {void}
             */
            handleScroll (el) {
                if ((el.offsetHeight + el.scrollTop) >= el.scrollHeight) {
                    this.loadMoreImages();
                }
            },

            /**
             * @return {void}
             */
            closeImgLib () {
                if (this.currentSegment === this.uploadImg && !isEmpty(this.currentImage)) {
                    this.deleteCurrentImg(this.currentImage.id);
                }

                this.clearLib();
                this.imageLibraryModalStatus = false;
            },

            /**
             * @param {string} imageUrl
             * @return {void}
             */
            async copyUrl (imageUrl) {
                copyTextToClipBoard(imageUrl);

                this.copyButtonDisabled = true;
                this.$message.show(this.trans('newsletter.url-copied-successfully'), 'success');
            },

            /**
             * @param {string} searchString
             * @param {number} count
             * @return {void}
             */
            async loadLibList (searchString = '', count = this.loadedCount) {
                this.imageLibraryLoadingStatus = true;
                let url = EndPoints.GET_AND_POST_IMAGE_IN_LIBRARY;

                if (count !== 0) {
                    url = `${EndPoints.GET_AND_POST_IMAGE_IN_LIBRARY}?count=${count}`;
                }

                if (!isEmptyString(searchString)) {
                    url = `${EndPoints.GET_AND_POST_IMAGE_IN_LIBRARY}?searchString=${searchString}`;
                }

                const res = await this.$http.get(url);

                this.imageLibList = res.data;
                this.imageLibraryLoadingStatus = false;
            },

            /**
             * @return {void}
             */
            clearLib () {
                this.uploadedFileInvalid = false;
                this.imageLoadError = false;
                this.corsError = false;
                this.loadedCount = Limits.LOAD_IMAGES_COUNT;
                this.imageFromLinkPreviewUrl = '';
                this.imageLibList = [];
                this.currentImage = {};
                this.imageUploadError = '';
            },

            /**
             * @return {void}
             */
            loadMoreImages () {
                this.loadedCount += Limits.LOAD_IMAGES_COUNT;
                this.loadLibList('', this.loadedCount);
            },

            /**
             * @param {number} id
             * @return {void}
             */
            selectImage (id) {
                this.copyButtonDisabled = false;
                this.currentImage = this.imageLibList.find(item => item.id === id);
                this.imgLibraryAcceptButtonDisabled = false;
            },

            /**
             * @param {number} imgId
             * @param {boolean} isFromLib
             * @param {boolean} isWhileSearch
             * @return {void}
             */
            async deleteCurrentImg (imgId, isFromLib = false, isWhileSearch = false) {
                this.imgLibraryAcceptButtonDisabled = true;
                this.imageLibraryLoadingStatus = true;

                if (imgId) {
                    this.currentImage = {};
                    const imgToDelete = this.imageLibList.indexOf(this.imageLibList.find(item => item.id === imgId));

                    this.imageLibList.splice(imgToDelete, 1);
                    await this.$http.delete(replaceString(EndPoints.DELETE_IMAGE, { id: imgId }));

                    if (!isFromLib && this.$refs.uploadArea) {
                        this.$refs.uploadArea.deleteImage();
                    }

                    if (this.currentSegment === this.imgLibrary && !isWhileSearch) {
                        this.loadLibList('', this.loadedCount);
                    }

                    this.imageLibraryLoadingStatus = false;
                }
            },

            /**
             * @param {Object} file
             * @return {void}
             */
            async imageUploaded (file) {
                const fileTypeOk = checkFileType(file, this.acceptedImageTypes);
                const fileSizeOk = file.size <= MAX_FILE_SIZE;

                this.copyButtonDisabled = false;
                this.imageLoadError = false;
                this.uploadedFileInvalid = false;

                if (fileTypeOk && fileSizeOk) {
                    this.uploadingStatus = true;

                    if (!isEmpty(this.currentImage)) {
                        this.deleteCurrentImg(this.currentImage.id);
                    }

                    const blob = new Blob([file], { type: file.type });
                    const formData = new FormData();

                    formData.append('image', blob, file.name);

                    const response = await this.$http.post(EndPoints.GET_AND_POST_IMAGE_IN_LIBRARY, formData);

                    if (response.status !== 201 || response.status !== 200) {
                        this.deleteCurrentImg();
                        this.imageLoadError = true;
                        this.imgLibraryAcceptButtonDisabled = true;
                    }

                    this.imgLibraryAcceptButtonDisabled = false;
                    this.currentImage = response.data;
                    this.uploadingStatus = false;

                    return;
                }

                this.imageUploadError = this.trans(!fileTypeOk
                    ? 'newsletter.invalid-image-message'
                    : 'newsletter.invalid-image-file-size'
                );

                this.$refs.uploadArea.deleteImage();
                this.uploadedFileInvalid = true;
            },

            /**
             * @param {object} currentImage
             * @return {void}
             */
            addImage (currentImage) {
                this.copyButtonDisabled = false;
                this.$emit('addImageToCodeEditor', currentImage.cloudUrl);
                this.clearLib();
            },

            /**
             * @param {blob} blob
             * @return {void}
             */
            async uploadFromLink (blob) {
                if (blob.size && blob.size > 0) {
                    const formData = new FormData();

                    formData.append('image', blob, this.currentImage.title);

                    const res = await this.$http.post(EndPoints.GET_AND_POST_IMAGE_IN_LIBRARY, formData);
                    this.currentImage = res.data;

                    // eslint-disable-next-line no-prototype-builtins
                    if (typeof res === 'object' && !!res.data && !res.data.hasOwnProperty('errors')) {
                        this.addImage(this.currentImage);
                    }
                }
            },

            /**
             * @param {string} url
             * @return {void}
             */
            async checkAndLoadImage (url) {
                this.copyButtonDisabled = false;
                this.corsError = false;
                this.isUrlInvalid = false;
                this.imageLoadError = false;

                if (isEmptyString(url)) {
                    this.clearLib();

                    return;
                }

                if (!url.match(VALID_URL_REGEXP)) {
                    this.isUrlInvalid = true;
                    this.clearLib();

                    return;
                }

                const response = await axios.get(url, { responseType: 'blob' });

                if (typeof response === 'object' && !!response.data) {
                    const blob = response.data;

                    if (!checkFileType(blob, this.acceptedImageTypes)) {
                        this.clearLib();
                        this.imageUploadError = this.trans('newsletter.invalid-image-message');

                        return;
                    }

                    if (blob.size > MAX_FILE_SIZE) {
                        this.clearLib();
                        this.imageUploadError = this.trans('newsletter.invalid-image-file-size');

                        return;
                    }

                    this.uploadedImageBlob = response.data;
                    this.currentImage = {
                        title: `${url.split('/').pop().split('#')[0].split('?')[0]}.${
                        this.uploadedImageBlob.type.split('/')[1]}`,
                        size: this.uploadedImageBlob.size,
                        cloudUrl: url
                    };
                    this.imageFromLinkPreviewUrl = url;
                    this.imgLibraryAcceptButtonDisabled = false;
                    this.isUrlInvalid = false;
                    this.corsError = false;
                    this.imageUploadError = '';

                    return;
                }

                this.clearLib();

                this.corsError = true;
            },

            /**
             * @return {void}
             */
            toggleAddImageModal () {
                this.imageLibraryModalStatus = !this.imageLibraryModalStatus;
            }
        }
    };
</script>

<style>
    .grid-rows-100 {
        grid-template-rows: max-content 1fr;
    }
    .image-info-offset {
        padding-bottom: 185px;
    }
</style>
