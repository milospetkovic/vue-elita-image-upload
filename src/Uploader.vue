<script>
    import Loader from './loader/index.vue';
    import axios from 'axios'

    export default {

        props:{
            server: {
                type: String,
                default: '/api/upload'
            },
            isInvalid: {
                type: Boolean,
                default: false
            },
            media:{
                type: Array,
                default: []
            },
            location:{
                type: String,
                default: ''
            },
            max:{
                type: Number,
                default: null
            },
            maxFilesize:{
                type: Number,
                default: 4
            },
            warnings:{
                type: Boolean,
                default: true
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
                isLoading:true
            }
        },
        expose: ['finishedRotateImage'],
        methods:{
            init(){
                this.addedMedia = this.media

                this.isLoading = true;

                this.getImagesPreview();
                this.renderImages();

                this.isLoading = false;

                this.$emit('init', this.allMedia)
            },
            async fileChange(event){
                this.isLoading = true
                let files = event.target.files

                for(var i=0; i < files.length; i++){
                    if(!this.max || this.allMedia.length < this.max){
                        if(files[i].size <= this.maxFilesize*1000000){
                            let formData = new FormData
                            let url = URL.createObjectURL(files[i])
                            formData.set('image', files[i])

                            const {data} = await axios.post(this.server, formData)

                            console.log('data from server', data);
                            let addedImage = {
                                id: data.data.id,
                                name: data.data.name,
                                size: files[i].size,
                                type: files[i].type,
                                url: this.location + "/" + data.data.name,
                            };
                            this.addedMedia.push(addedImage)

                            this.$emit('change', this.allMedia)
                            this.$emit('add', addedImage, this.addedMedia)
                            this.renderImages()
                        }else{
                            this.$emit('maxFilesize', files[i].size)
                            if(this.warnings){
                                alert('The file you are trying to upload is too big. \nMaximum Filesize: '+ this.maxFilesize +'MB')
                            }
                            break;
                        }
                    }else{
                        this.$emit('max')
                        if(this.warnings){
                            alert('You have reached the maximum number of files that you can upload. \nMaximum Files: '+ this.max)
                        }
                        break;
                    }
                }
                event.target.value = null
                this.isLoading = false
            },
            removeAddedMedia(index){
                let removedImage = this.addedMedia[index]
                this.addedMedia.splice(index,1)

                this.$emit('change', this.allMedia)
                this.$emit('remove', removedImage, this.removedMedia)
            },
            renderImages() {
                console.log('renderimages ');
                // Koristimo this.$nextTick() kako bismo osigurali da se refs objekat ažurira
                this.$nextTick(() => {
                    this.addedMedia.forEach((image, index) => {
                        const canvas = this.$refs['canvas' + index][0];
                        const ctx = canvas.getContext('2d');

                        // Kreiramo novu sliku
                        const img = new Image();
                        img.onload = () => {

                            canvas.width = 140; // Postavite željenu širinu
                            canvas.height = 90; // Postavite željenu visinu

                            // Postavljanje antialiasing-a
                            ctx.imageSmoothingEnabled = true;
                            ctx.imageSmoothingQuality = 'high';

                            // Postavljanje dimenzija i položaja slike unutar canvasa
                            const scale = Math.min(canvas.width / img.width, canvas.height / img.height);
                            const scaledWidth = img.width * scale;
                            const scaledHeight = img.height * scale;
                            const offsetX = (canvas.width - scaledWidth) / 2;
                            const offsetY = (canvas.height - scaledHeight) / 2;

                            // Crtanje slike na canvasu
                            ctx.drawImage(img, offsetX, offsetY, scaledWidth, scaledHeight);
                        };
                        img.src = image.url;
                    });
                });
            },
            rotateImage(index, rotateLeft) {

                console.log('in component rotateLeft', rotateLeft);

                let urlsByIndex = [];

                this.$emit('rotateImage', this.addedMedia[index], rotateLeft);

                const canvas = this.$refs['canvas' + index][0];
                const ctx = canvas.getContext('2d');
                const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);

                // Rotiranje dimenzija canvasa
                const temp = canvas.width;
                canvas.width = canvas.height;
                canvas.height = temp;

                // Postavljanje antialiasing-a
                ctx.imageSmoothingEnabled = true;
                ctx.imageSmoothingQuality = 'high';

                // Centriranje slike u novim dimenzijama canvasa
                const offsetX = (canvas.width - imgData.height) / 2;
                const offsetY = (canvas.height - imgData.width) / 2;

                // Rotiranje slike za 90 stepeni ulevo i crtanje na canvasu
                ctx.putImageData(this.rotateImageData(imgData), offsetX, offsetY);
            },
            rotateRight(index) {

                // Rotiranje slike za 90 stepeni ulevo
                const canvas = this.$refs['canvas' + index][0];
                const ctx = canvas.getContext('2d');
                const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);

                canvas.width = imgData.height;
                canvas.height = imgData.width;
                ctx.putImageData(this.rotateImageData(imgData), 0, 0);
            },
            rotateImageData(imgData) {
                const newImgData = new ImageData(imgData.height, imgData.width);

                for (let x = 0; x < imgData.width; x++) {
                    for (let y = 0; y < imgData.height; y++) {
                        const newIndex = (x * imgData.height + (imgData.height - y)) * 4;
                        const oldIndex = (y * imgData.width + x) * 4;

                        newImgData.data[newIndex] = imgData.data[oldIndex];
                        newImgData.data[newIndex + 1] = imgData.data[oldIndex + 1];
                        newImgData.data[newIndex + 2] = imgData.data[oldIndex + 2];
                        newImgData.data[newIndex + 3] = imgData.data[oldIndex + 3];
                    }
                }

                return newImgData;
            },
            getImagesPreview() {
                this.addedMedia.forEach((image, index) => {
                    if (!this.addedMedia[index].url) {
                        this.addedMedia[index].url = this.location + "/" + image.name
                    }
                });
            },
            finishedRotateImage(response) {
                console.log('finishedRotateImage', response);
            },
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
        ],
        components:{
            Loader
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
                <div class="mu-plusbox-container">
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
                <div v-for="(image, index) in addedMedia" :key="index" class="mu-image-container">
                    <canvas :ref="'canvas' + index" class="mu-images-preview"></canvas>
                    <button
                            @click="removeAddedMedia(index)"
                            :disabled="buttonsDisabled"
                            class="mu-close-btn"
                            type="button"
                    >
                        <i class="mdi mdi-delete text-xl"></i>
                    </button>
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
    </div>
</template>
<style scoped>
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
        width: 140px !important;
        margin: 0.25rem !important;
    }
    .mu-plusbox {
        background-color: #ffffff !important;
        border: 1px dashed #818181 !important;
        border-radius: 5px !important;
        cursor: pointer !important;
        display: flex !important;
        flex-wrap: wrap !important;
        align-items: center !important;
        width: 140px !important;
        height: 90px !important;
    }
    .mu-plusbox:hover{
        background-color: #f1f1f1 !important;
    }
    .mu-plusbox:hover > .mu-plus-icon{
        color: #028296 !important;
    }
    .mu-plus-icon{
        color: #00afca !important;
        font-size: 3rem !important;
        flex: 1;
    }
    .mu-image-container{
        width: 140px !important;
        height: 90px !important;
        margin: 0.25rem !important;
        position: relative;
    }
    .mu-images-preview {
        border-radius: 5px !important;
        border: 1px solid #818181 !important;
        width: 140px !important;
        height: 90px !important;
        transition: filter 0.1s linear;
        object-fit: cover;
        object-position: center;
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
        top: 0px;
        right: 3px;
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
        top: 20px;
        right: 3px;
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
        top: 40px;
        right: 3px;
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
</style>