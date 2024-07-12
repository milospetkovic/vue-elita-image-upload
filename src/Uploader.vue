<script>
    import Loader from './loader/index.vue';
    import axios from 'axios'

    export default {

        props:{
            server: {
                type: String,
                default: '/api/upload',
            },
            isInvalid: {
                type: Boolean,
                default: false,
            },
            media:{
                type: Array,
                default: [],
            },
            location:{
                type: String,
                default: '',
            },
            max:{
                type: Number,
                default: null,
            },
            maxFilesize:{
                type: Number,
                default: 20,
            },
            warnings:{
                type: Boolean,
                default: true,
            },
            showLeftRotateButton:{
                type: Boolean,
                default: false,
            },
            showRightRotateButton:{
                type: Boolean,
                default: false,
            },
            buttonsDisabled: {
                type: Boolean,
                default: false,
            },
        },
        mounted() {
            this.init()
        },
        data(){
            return{
                addedMedia:[],
                removedMedia:[],
                isLoading:true,
                dragIndex: null,
                dropIndex: null,
                mouseX: 0,
                mouseY: 0,
            }
        },
        expose: [
            'finishedRotateImage',
            'finishedDeleteImage',
            'finishedSortImages',
        ],
        methods:{
            init(){
                this.addedMedia = this.media;

                this.isLoading = true;

                this.getImagesPreview();
                this.renderImages();

                this.isLoading = false;

                this.$emit('init', this.allMedia)
            },
            onImageDragStart(index) {
                this.dragIndex = index;
            },
            onImageDrop(index) {
                if (this.dragIndex !== null) {
                    this.dropIndex = index;
                    // Splice to move the dragged element
                    const draggedElement = this.addedMedia.splice(this.dragIndex, 1)[0];
                    this.addedMedia.splice(this.dropIndex, 0, draggedElement);
                    this.dragIndex = null;
                    this.dropIndex = null;
                }
            },
            onImageDragEnd() {
                // show loader.
                this.isLoading = true;

                // Collect sorted ids.
                let ids = [];
                this.addedMedia.forEach(item => {
                    ids.push(item.id);
                });

                this.$emit('imagesSort', ids);
            },
            dragOver(e) {
                e.preventDefault();

            },
            dragLeave(e) {
                e.preventDefault();
            },
            drop(e) {
                e.preventDefault();
                this.handleDropFiles(e);
            },
            handleDropFiles(event) {
                event.target.files = event.dataTransfer.files;
                this.fileChange(event)
            },
            async fileChange(event) {
                this.isLoading = true
                let files = event.target.files;

                for(var i=0; i < files.length; i++){
                    if(!this.max || this.allMedia.length < this.max){
                        if(files[i].size <= this.maxFilesize*1000000){
                            let formData = new FormData
                            let url = URL.createObjectURL(files[i])
                            formData.set('image', files[i])

                            // Get data from the server.
                            const {data} = await axios.post(this.server, formData);

                            if (!data.isError) {

                                let addedImage = {
                                    id: data.data.id,
                                    name: data.data.name,
                                    size: files[i].size,
                                    type: files[i].type,
                                    url: this.location +
                                            "/thumbs" +
                                            "/" + data.data.id +
                                            "/" + data.data.fname +
                                            "_130x100." + data.data.fext,
                                };
                                this.addedMedia.push(addedImage)

                                this.$emit('change', this.allMedia)
                                this.$emit('add', addedImage, this.addedMedia)
                                this.renderImages()
                            } else {
                                console.error('Error from server', data.errorMessage);
                                this.showUploadError(data.errorMessage);
                            }
                        }else{
                            this.$emit('maxFilesize', files[i].size)
                            if(this.warnings){
                                alert('Slika je prevelika. \nMaksimalna dozvoljena veličina slike je: '+ this.maxFilesize +'MB')
                            }
                            break;
                        }
                    }else{
                        this.$emit('max')
                        if(this.warnings){
                            alert('Dostigli ste maksimalan broj slika koje možete da postavite. \nMaksimalan dozvoljen broj je: '+ this.max)
                        }
                        break;
                    }
                }
                event.target.value = null
                this.isLoading = false
            },
            removeAddedMedia(index){
                let removedImage = this.addedMedia[index]
                this.$emit('change', this.allMedia)
                this.$emit('remove', removedImage, this.removedMedia)
            },
            renderImages() {
                this.$nextTick(() => {
                    this.addedMedia.forEach((image, index) => {
                        const canvas = this.$refs['canvas' + index][0];
                        const imageSrc = encodeURI(this.addedMedia[index].url);
                        this.drawImageOnCanvas(canvas, imageSrc);
                    });
                });
            },
            drawImageOnCanvas(canvas, imageSrc) {
                const ctx = canvas.getContext('2d');

                // Clear canvas before drawing
                ctx.clearRect(0, 0, canvas.width, canvas.height);

                const img = new Image();

                img.onload = () => {
                    //canvas.width = 140;
                    //canvas.height = 90;

                    const scale = Math.min(canvas.width / img.width, canvas.height / img.height);
                    const scaledWidth = img.width * scale;
                    const scaledHeight = img.height * scale;
                    const offsetX = (canvas.width - scaledWidth) / 2;
                    const offsetY = (canvas.height - scaledHeight) / 2;

                    ctx.clearRect(0, 0, canvas.width, canvas.height);
                    ctx.drawImage(img, offsetX, offsetY, scaledWidth, scaledHeight);
                };
                img.src = imageSrc + '?t=' + new Date().getTime();
            },
            rotateImage(index, rotateLeft=false) {
                this.$emit('rotateImage', this.addedMedia[index], index, rotateLeft);
            },
            getImagesPreview() {
                this.addedMedia.forEach((image, index) => {
                    if (!this.addedMedia[index].url) {
                        this.addedMedia[index].url = this.location +
                                "/thumbs" +
                                "/" + image.id +
                                "/" + image.fname +
                                "_130x100." + image.fext;
                    }
                });
            },
            finishedRotateImage(response, index) {
                if (response.data.isError === false) {
                    const canvas = this.$refs['canvas' + index][0];
                    const imageSrc = encodeURI(this.addedMedia[index].url);
                    this.drawImageOnCanvas(canvas, imageSrc);
                } else {
                    //
                }
            },
            finishedDeleteImage(response) {
                if (response.data.isError === false) {
                    let imgId = response.data.data[0].id;
                    this.addedMedia.forEach((image, index) => {
                        if (image.id === imgId) {
                            this.addedMedia.splice(index,1);
                            this.renderImages();
                            return true;
                        }
                    });
                } else {
                    //
                }
            },
            finishedSortImages(response) {
                if (response.data.isError === false) {
                    this.renderImages();
                } else {
                    //
                }
                this.isLoading = false;
            },
            showUploadError(message) {
                const errorDiv = document.createElement('div');
                errorDiv.classList.add('text-red');
                errorDiv.textContent = message;
                const container = document.getElementById('upload-errors');
                container.appendChild(errorDiv);
                setTimeout(() => {
                    container.removeChild(errorDiv);
                }, 5000);
            }
        },
        computed:{
            allMedia(){
                return [...this.addedMedia];
            },
        },
        emits: [
            'init',
            'change',
            'add',
            'remove',
            'max',
            'maxFilesize',
            'rotateImage',
            'imagesSort',
        ],
        components: {
            Loader, 
        }
    }
</script>
<template>
    <div>
        <div class="mu-container" :class="isInvalid?'mu-red-border':''">
            <Loader
                color="#0275d8"
                :active="isLoading"
                spinner="line-scale"
                background-color = 'rgba(255, 255, 255, .4)'
            />
            <div class="mu-elements-wraper">

                <!--UPLOAD BUTTON-->
                <div class="mu-plusbox-container"
                     @dragover.prevent="dragOver"
                     @dragleave.prevent="dragLeave"
                     @drop.prevent="drop"
                >
                    <label for="mu-file-input" class="mu-plusbox">
                        <svg
                            class="mu-plus-icon"
                            xmlns="http://www.w3.org/2000/svg"
                            width="1em"
                            height="1em"
                            preserveAspectRatio="xMidYMid meet"
                            viewBox="0 0 24 24">
                            <g fill="none">
                                <path
                                    d="M12 1C5.925 1 1 5.925 1 12s4.925 11 11 11s11-4.925 11-11S18.075 1 12 1zm1 15a1 1 0 1 1-2 0v-3H8a1 1 0 1 1 0-2h3V8a1 1 0 1 1 2 0v3h3a1 1 0 1 1 0 2h-3v3z"
                                    fill="currentColor"/>
                            </g>
                        </svg>
                    </label>
                    <input @change="fileChange" id="mu-file-input" type="file" accept="image/*" multiple hidden>
                </div>

                <!--IMAGES PREVIEW-->
                <div
                        v-for="(image, index) in addedMedia"
                        :key="index"
                        class="mu-image-container"
                        :draggable="true"
                        @dragover.prevent
                        @drop="onImageDrop(index)"
                        @dragstart="onImageDragStart(index)"
                        @dragend="onImageDragEnd"
                        @touchstart="onImageDragStart(index)"
                        @touchend="onImageDragEnd"
                >
                    <canvas :ref="'canvas' + index" class="mu-images-preview"></canvas>
                    <div class="img-actions-box">
                        <template v-if="showLeftRotateButton">
                            <button
                                @click="rotateImage(index, true)"
                                :disabled="buttonsDisabled"
                                class="mu-left-rotate-btn"
                                type="button"
                            >
                                <i class="mdi mdi-arrow-u-left-top-bold text-xl"></i>
                            </button>
                        </template>
                        <template v-if="showRightRotateButton">
                            <button
                                @click="rotateImage(index)"
                                :disabled="buttonsDisabled"
                                class="mu-right-rotate-btn"
                                type="button"
                            >
                                <i class="mdi mdi-arrow-u-right-top-bold text-xl"></i>
                            </button>
                        </template>

                        <button
                            @click="removeAddedMedia(index)"
                            :disabled="buttonsDisabled"
                            class="mu-close-btn"
                            type="button"
                        >
                            <i class="mdi mdi-delete text-xl"></i>
                        </button>
                    </div>
                </div>
            </div>
        </div>
        <div>
            <div v-for="(image, index) in addedMedia" :key="index" class="mu-mt-1">
                <input type="text" name="added_media[]" :value="image.name" hidden>
            </div>
            <div v-for="(image, index) in removedMedia" :key="index" class="mu-mt-1">
                <input type="text" name="removed_media[]" :value="image.name" hidden>
            </div>
            <div v-if="allMedia.length" class="mu-mt-1">
                <input type="text" name="media" value="1" hidden>
            </div>
        </div>
        <div id="upload-errors">
        </div>
    </div>
</template>
<style>
    .mu-container{
        background-color: #fbfbfb !important;
        border-radius: 5px !important;
        border-style: solid !important;
        border: 1px solid #9b9b9b !important;
        box-sizing: border-box !important;
        width: 100% !important;
        height: auto !important;
    }
    .mu-elements-wraper {
        padding: 1rem !important;
        display: flex !important;
        flex-wrap: wrap !important;
    }
    .mu-plusbox-container{
        display: inline-flex !important;
        height: 90px !important;
        width: 100% !important;
        margin: 0.25rem !important;
    }
    .mu-plusbox {
        background-color: #f1f1f1 !important;
        border: 1px dashed #818181 !important;
        border-radius: 5px !important;
        cursor: pointer !important;
        display: flex !important;
        flex-wrap: wrap !important;
        align-items: center !important;
        width: 100% !important;
        height: 90px !important;
    }
    .mu-plusbox:hover{
        background-color: #f6f6f6 !important;
    }
    .mu-plusbox:hover > .mu-plus-icon{
        color: #028296 !important;
    }
    .mu-plus-icon{
        color: #00afca !important;
        font-size: 3rem !important;
        flex: 1;
    }
    .mu-image-container {
        width: 140px !important;
        height: 90px !important;
        margin: 0.25rem !important;
        position: relative;
        margin-right: 1.5rem !important;
        transition: transform 0.2s ease-in-out;
    }
    .mu-image-container:hover {
        cursor: crosshair;
    }
    .mu-images-preview {
        border-radius: 5px !important;
        border: 1px solid #818181 !important;
        width: 140px !important;
        height: 90px !important;
        /*transition: filter 0.1s linear;*/
        object-fit: cover;
        object-position: center;
        background: gray;
    }
    .mu-images-preview:hover{
        filter: brightness(90%);
    }
    button:disabled {
        opacity: 0.4;
    }
    .mu-close-btn{
        display:block;
        background: none !important;
        color:white !important;
        border: none !important;
        padding: 0px !important;
        margin:0px !important;
        cursor: pointer !important;
        position: absolute !important;
        top: 60px;
        right: 1px;
    }
    .mu-left-rotate-btn {
        display:block;
        background: none !important;
        color:white !important;
        border: none !important;
        padding: 0px !important;
        margin:0px !important;
        cursor: pointer !important;
        position: absolute !important;
        top: 0px;
        right: 1px;
    }
    .mu-right-rotate-btn {
        display:block;
        background: none !important;
        color:white !important;
        border: none !important;
        padding: 0px !important;
        margin:0px !important;
        cursor: pointer !important;
        position: absolute !important;
        top: 22px;
        right: 1px;
    }
    .mu-times-icon{
        font-size: 3rem !important;
        filter: drop-shadow(0px 0px 1px black);
    }
    .mu-close-btn:hover{
        color: red !important;
    }
    .mu-left-rotate-btn:hover,
    .mu-right-rotate-btn:hover {
        color: yellow !important;
    }
    .mu-red-border {
        border: 1px solid #dc3545 !important;
    }
    .mu-mt-1 {
        margin-top: 0.25rem !important;
    }
    img {
        -webkit-user-drag: none;
        -khtml-user-drag: none;
        -moz-user-drag: none;
        -o-user-drag: none;
    }
    .text-red {
        color: red;
    }
    .img-actions-box {
        position: absolute;
        right: -23px;
        top: 0;
        width: 22px;
        height: 90px;
        background: gray;
        border-radius: 5px !important;
    }
</style>