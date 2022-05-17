<template>
  <div class="q-pa-md">
    <q-stepper
      v-model="step"
      ref="stepper"
      color="primary"
      contracted
    >
      <q-step
        v-for="(question, objKey, qnb) in questions"
        :key=qnb
        :name="qnb+1"
        :title="objKey"
        :prefix="objKey"
        :done="step > qnb"
        :error="question.answers.length>2"
      >
      <q-slide-transition>
        <div v-show="(state.willPlay[objKey]||0)>=0*question.all.length" class="row justify-center q-pb-lg">
          <q-slide-transition>
            <div class="col-12 text-center text-red q-pa-none" v-show="question.answers.length>2">Ne sélectionnez pas plus 2 réponses</div>
          </q-slide-transition>
          <div class="col-12 text-center q-pa-md">Quels échantillons audio pensez-vous entendre quand vous cliquez sur le gros bouton rond ?</div>
          <q-btn :disabled="state.hasPlayedDichotic[objKey]" icon="play_arrow" @click="playBoth(question.left, question.right,objKey)" :loading="state.playing&&state.isPlayingDichotic[objKey]" round size="lg"/>
        </div>
      </q-slide-transition>
      <div class="row no-wrap justify-center col-12">
        <div class="column items-center q-mx-sm" v-for="(trial, index) in question.all" :key=index>
          <q-btn :disabled="(state.willPlay[objKey]||0)!==index||!(state.hasPlayedDichotic[objKey]||false)" @click="change(objKey,index)">
            <div class="row no-wrap">
              <div class="col-6" :style="{position: 'relative', transition: 'all 1s', overflow: 'hidden', maxWidth: (state.willPlay[objKey]||0)===index?'3em':'0em'}">
                <q-spinner :style="{transition: 'all 0.2s', opacity:state.playing&&!state.isPlayingDichotic[objKey]?1:0}"/>
                <q-icon style="position: absolute; top:0px; left:0px; }" name="play_arrow" :style="{transition: 'all 0.2s', opacity:state.playing&&!state.isPlayingDichotic[objKey]?0:1}"/>
              </div>
              <div class="col-6">
                <span class="col text-center q-px-sm">{{String.fromCharCode(parseInt(index,10)+65)}}</span>
              </div>
            </div>
          </q-btn>
          <div :style="{overflow:'hidden', transition: 'all 1s', maxHeight: (state.willPlay[objKey]||0)>=question.all.length||state.hasPlayedDichotic[objKey]?'100%':'0%'}">
              <q-checkbox size="lg" keep-color v-model="question.answers" :val="trial" :color="state.showAnswers&&(question.left===trial || question.right==trial)?'green':''"/>
          </div>

        </div>
      </div>
      
      <q-toggle
        v-model="state.showAnswers"
        label="Montrer les réponses"
        color="green"
      />


        <q-stepper-navigation>
          <q-btn @click="$refs.stepper.next()" color="primary" :disabled="question.answers.length>2" :label="step === Object.keys(questions).length ? 'Finish' : 'Continue'" />
          <q-btn v-if="step > 1" flat color="primary" @click="$refs.stepper.previous()" label="Back" class="q-ml-sm" />
        </q-stepper-navigation>

      </q-step>




    </q-stepper>
  </div>
</template>

<script>
import { defineComponent, ref, reactive, onMounted } from 'vue'
import buttonSfx from '../assets/crotte.mp3'
import { useSound } from '@vueuse/sound'
import 'lodash.product'
import { chunk,clone,random, product,filter,flattenDeep, includes, compact, differenceWith, without, shuffle, concat, uniq, range, times, unzip, flatten, constant, zip, uniqWith, isEqual, reverse, sampleSize } from 'lodash';
export default defineComponent({
  setup() {
    const playing=ref(false)
    const questions = reactive({
      1:{"left": "English_S02_M01_1",
      "right": "English_S02_M02_1",
      "all": ["English_S02_M03_1","English_S02_M04_1","English_S02_M01_1","English_S02_M02_1"],
      "answers": []
      },
      2:{"left": "English_S02_M01_1",
        "right": "English_S02_M02_1",
        "all": ["English_S02_M03_1","English_S02_M04_1","English_S02_M01_1","English_S02_M02_1"],
        "answers": []
      },
      3:{"left": "English_S02_M01_1",
        "right": "English_S02_M02_1",
        "all": ["English_S02_M03_1","English_S02_M04_1","English_S02_M01_1","English_S02_M02_1"],
        "answers": []
      }
    })
    const answers=reactive(Object.fromEntries(Object.keys(questions).map(k => [k, []])))
    const state=reactive({currentQuestion:undefined, showAnswers:false, playing: false, willPlay: {}, hasPlayedDichotic: {}, isPlayingDichotic:{}})
    const generalSpecs={minMelId:0, maxMelId:9, minTxtId:1, maxTxtId:4}//mettre des nombres inférieurs de 1 aux valeurs souhaitées
    let availableOptions=shuffle(product(range(generalSpecs.minMelId,generalSpecs.maxMelId+1),range(generalSpecs.minTxtId,generalSpecs.maxTxtId+1)))
    const generateDichoticSets=(nbSets,minId,maxId)=>{
      //les limites sont inclusives !
      let leftSet, rightSet, dichoticSets, finalSets
      do {
        //leftSet=times(nbSets*2, random.bind(minId, constant(10))) TIRAGE ALEATOIRE FAUSSÉ, BUG DANS LODASH ?
        leftSet=Array.from({length: nbSets*2}, () => random(minId,maxId))
        rightSet=Array.from({length: nbSets*2}, () => random(minId,maxId))
        dichoticSets=sampleSize(uniqWith(unzip([leftSet,rightSet]).filter(item=>item[0]!==item[1]),isEqual),nbSets) //tire un tableau de 10 elements uniques qui associent chacun deux melodies différentes
        //console.log("dichotic", dichoticSets)
        finalSets=zip(...dichoticSets) //replace les melodies dans deux tableaux ordonnés séparés (un pour gauche l'autre pour droite)
        //console.log('finalSets', finalSets)
        //console.log('essai sur les Sets')
      } while (dichoticSets.length<nbSets) 
      // pour associer les deux unzip([leftMel,rightMel])
      //console.log({dichoticSets:dichoticSets, leftSet:finalSets[0], rightSet:finalSets[1]})
      
      return({dichoticSets:dichoticSets, leftSet:finalSets[0], rightSet:finalSets[1]})
    }
    const generateReverseSamples=(Sets,nbReverse)=>{
      let sample=sampleSize(Sets,nbReverse)
      sample.forEach((item,index)=>{
        sample[index] = [...item].reverse()
        }
      )

      return sample
    }
    const generateDichoticSamples = (nbSamples, forbidDoublons, symetrical='none', specs=generalSpecs) =>{
      //cette fonction est tordue et exagerement compliquée. un fonctionnement apartir de generateOptions aurait été bien plus lisible
      let fullDichotSamples
      do {
        let leftDichotTxts,rightDichotTxts,rightDichotMels,leftDichotMels
        if (symetrical==='mel'){

          rightDichotMels=sampleSize(range(specs.minMelId, specs.maxMelId+1),nbSamples)
          leftDichotMels=rightDichotMels
        } else {
          ({leftSet:leftDichotMels,rightSet:rightDichotMels}=generateDichoticSets(nbSamples,specs.minMelId,specs.maxMelId))
        }
      
        if (symetrical==='text'){
          rightDichotTxts=sampleSize(range(specs.minTxtId, specs.maxTxtId+1),nbSamples)
          leftDichotTxts=rightDichotTxts
        } else {
          ({leftSet:leftDichotTxts,rightSet:rightDichotTxts}=generateDichoticSets(nbSamples,specs.minTxtId,specs.maxTxtId))
        }
        const leftDichotSamples=unzip([leftDichotMels,leftDichotTxts])
        const rightDichotSamples=unzip([rightDichotMels,rightDichotTxts])
        console.log('cestunessai',symetrical)
        fullDichotSamples=unzip([leftDichotSamples,rightDichotSamples])
      } while (forbidDoublons&&uniqWith(flatten(fullDichotSamples),isEqual).length<flatten(fullDichotSamples).length)
      availableOptions=differenceWith(availableOptions,flatten(fullDichotSamples), isEqual)
      return fullDichotSamples
    }
    const generateTextSymetricSamples=(nbSamples)=>{
      let fixedId, samples
      let allSamples=[]
      let impossible=false
      for(let i=0;i<nbSamples;i++){
        let counter=0
        do{
          fixedId=random(generalSpecs.minTxtId,generalSpecs.maxTxtId)
          console.log('fixed text',fixedId)
          samples=sampleSize(availableOptions.filter(item=>item[1]===fixedId),4)
          counter++
        } while (samples.length<4&&counter<100)
        if (counter>=100){
          console.log("impossible")
          impossible=true
          }
        
        samples=concat([chunk(samples,2)[0]],[shuffle(samples)])
        allSamples=concat(allSamples,[samples])

        availableOptions=differenceWith(availableOptions,chunk(flattenDeep(allSamples),2), isEqual)

      }
      console.log('coucou text',allSamples)
      return impossible ? undefined:allSamples
    }
    const generateMelSymetricSamples=(nbSamples)=>{
      let fixedId, samples
      let allSamples=[]
      let impossible=false
      for(let i=0;i<nbSamples;i++){
        let counter=0
        do{
          fixedId=random(generalSpecs.minMelId,generalSpecs.maxMelId)
          samples=sampleSize(availableOptions.filter(item=>item[0]===fixedId),4)
          counter++
        } while (samples.length<4&&counter<100)
        if (counter>=100){
          console.log("impossible")
          impossible=true
        }
        samples=concat([chunk(samples,2)[0]],[shuffle(samples)])
        allSamples=concat(allSamples,[samples])

        availableOptions=differenceWith(availableOptions,chunk(flattenDeep(allSamples),2), isEqual)

      }
      console.log('coucou memm',allSamples)
      return impossible ? undefined:allSamples
    }

    const generateOptions=(answers)=>{
      console.log('available',availableOptions.length)
      let impossible,list,internalAvailableOptions
      //let allOptions=product(range(specs.minMelId,specs.maxMelId+1),range(specs.minTxtId,specs.maxTxtId+1))//le probleme est là
      //allOptions=differenceWith(allOptions,flatten(answers),isEqual)
      let counter2=0
      do{ //ce do while ne sert a rien car impossible est undefined et non pas false
        list=clone(answers)
        internalAvailableOptions=clone(availableOptions)
        list.forEach((answer,index)=>{
          let tempList, option,mels,texts,tempAvailableOptions
          let counter=0
          do{
            tempAvailableOptions=clone(internalAvailableOptions)
            tempList=clone(answer)
            for(let i=0;i<2;i++){
              mels=unzip(tempList)[0]
              texts=unzip(tempList)[1]
              option=tempAvailableOptions.find(item=>!includes(mels,item[0])&&!includes(texts,item[1]))
              tempAvailableOptions=differenceWith(tempAvailableOptions,[option],isEqual)
              tempList=concat(tempList,[option])
            }
            counter++
          } while (option===undefined&&counter<=100)
          if (counter>=100){
            console.log('option impossible')
            impossible=true
          } else {
            internalAvailableOptions=differenceWith(internalAvailableOptions,concat([option],tempList),isEqual)
            list[index]=[chunk(tempList,2)[0],shuffle(tempList)]
          }

        })
        counter2++
        console.log('counter2',counter2)
      }while (impossible===false&&counter2<=200)//ce do while est une erreur sans gravité: il ne sert a rien car impossible est undefined et non pas false
      if (counter2>=200||impossible){
        console.log('grosse option impossible')
        return undefined
      }else{
        availableOptions=differenceWith(availableOptions,chunk(flattenDeep(list),2),isEqual)
        list.forEach((item,index)=>{
        })
        return list
      }
    }


    const convertArrayToId=(array)=>{
      return('English_S'+String(array[1]+1).padStart(2,'0')+'_M'+String(array[0]+1).padStart(2,'0')+'_1')
    }

    const generateList=()=>{
      let fullList,fullDichotSamples,someInvertedSamples,textSymetricSamples,melSymetricSamples
      do{
        console.log('essaiGeneral')
        availableOptions=shuffle(product(range(generalSpecs.minMelId,generalSpecs.maxMelId+1),range(generalSpecs.minTxtId,generalSpecs.maxTxtId+1)))
        console.log(product(range(generalSpecs.minMelId,generalSpecs.maxMelId+1),range(generalSpecs.minTxtId,generalSpecs.maxTxtId+1)))
        fullDichotSamples=generateDichoticSamples(2,true)
        textSymetricSamples=generateTextSymetricSamples(2)
        melSymetricSamples=generateMelSymetricSamples(2)
        console.log('mel',melSymetricSamples)
        someInvertedSamples=generateReverseSamples(fullDichotSamples,2)
        fullDichotSamples=generateOptions(fullDichotSamples)
        someInvertedSamples=generateOptions(someInvertedSamples)
        fullList=shuffle(concat(fullDichotSamples,someInvertedSamples,melSymetricSamples,textSymetricSamples))

      } while (includes(fullList,undefined))
      console.log('essai general',fullList.filter(item=>includes(item[1],...item[0])))
      console.log(chunk(flattenDeep(fullList),2).length,uniqWith(chunk(flattenDeep(fullList),2),isEqual).length)
      console.log(fullList)
      let flattenedList=chunk(flattenDeep(fullList),2)
      flattenedList.forEach((item,index)=>{flattenedList[index]=convertArrayToId(item)})
      flattenedList=chunk(flattenedList,6)
      flattenedList.forEach((item,index)=>{
        questions[index+1]={'left':item[0],'right':item[1],'all':[item[2],item[3],item[4],item[5]],'answers':[]}
        })
      console.log(flattenedList)

    }
    onMounted(() => {
         generateList()
       })
    const spritemap = {
    "English_S01_M01_1": [
      3000, //0
      100 //6000
    ],
    "English_S01_M06_1": [
      7000,
      6000
    ],
    "English_S01_M07_1": [
      14000,
      6000
    ],
    "English_S01_M08_1": [
      21000,
      6000
    ],
    "English_S01_M09_1": [
      28000,
      6000
    ],
    "English_S01_M10_1": [
      35000,
      6000
    ],
    "English_S02_M01_1": [
      42000,
      6000
    ],
    "English_S02_M02_1": [
      49000,
      6000
    ],
    "English_S02_M03_1": [
      56000,
      6000
    ],
    "English_S02_M04_1": [
      63000,
      6000
    ],
    "English_S02_M05_1": [
      70000,
      6000
    ],
    "English_S02_M06_1": [
      77000,
      6000
    ],
    "English_S02_M07_1": [
      84000,
      6000
    ],
    "English_S02_M08_1": [
      91000,
      6000
    ],
    "English_S02_M09_1": [
      98000,
      6000
    ],
    "English_S02_M10_1": [
      105000,
      6000
    ],
    "English_S03_M01_1": [
      112000,
      6000
    ],
    "English_S03_M02_1": [
      119000,
      6000
    ],
    "English_S03_M03_1": [
      126000,
      6000
    ],
    "English_S03_M04_1": [
      133000,
      6000
    ],
    "English_S03_M05_1": [
      140000,
      6000
    ],
    "English_S03_M06_1": [
      147000,
      6000
    ],
    "English_S03_M07_1": [
      154000,
      6000
    ],
    "English_S03_M08_1": [
      161000,
      6000
    ],
    "English_S03_M09_1": [
      168000,
      6000
    ],
    "English_S03_M10_1": [
      175000,
      6000
    ],
    "English_S04_M01_1": [
      182000,
      6000
    ],
    "English_S04_M02_1": [
      189000,
      6000
    ],
    "English_S04_M03_1": [
      196000,
      6000
    ],
    "English_S04_M04_1": [
      203000,
      6000
    ],
    "English_S04_M05_1": [
      210000,
      6000
    ],
    "English_S04_M06_1": [
      217000,
      6000
    ],
    "English_S04_M07_1": [
      224000,
      6000
    ],
    "English_S04_M08_1": [
      231000,
      6000
    ],
    "English_S04_M09_1": [
      238000,
      6000
    ],
    "English_S04_M10_1": [
      245000,
      6000
    ],
    "English_S05_M01_1": [
      252000,
      6000
    ],
    "English_S05_M02_1": [
      259000,
      6000
    ],
    "English_S05_M03_1": [
      266000,
      6000
    ],
    "English_S05_M04_1": [
      273000,
      6000
    ],
    "English_S05_M05_1": [
      280000,
      6000
    ],
    "English_S05_M06_1": [
      287000,
      6000
    ],
    "English_S05_M07_1": [
      294000,
      6000
    ],
    "English_S05_M08_1": [
      301000,
      6000
    ],
    "English_S05_M09_1": [
      308000,
      6000
    ],
    "English_S05_M10_1": [
      315000,
      6000
    ],
    "English_S06_M01_1": [
      322000,
      6000
    ],
    "English_S06_M02_1": [
      329000,
      6000
    ],
    "English_S06_M03_1": [
      336000,
      6000
    ],
    "English_S06_M04_1": [
      343000,
      6000
    ]
  }
    const onStartPlaying = () => {
      console.log("cajoue")
      state.playing=true
    }
    const onEndAny = (isStereo) => {
      console.log(state.willPlay)
      state.playing=false
      state.isPlayingDichotic[state.currentQuestion]=false
      }
    const onEndStereo = () => {
      onEndAny()
      state.willPlay[state.currentQuestion]=state.currentQuestion?(state.willPlay[state.currentQuestion]||0)+1:state.willPlay[state.currentQuestion] //cette condition est fausse tant qu'il n'y a pas eu de clic de l'utilisateur
    }
    const playStereo = useSound(buttonSfx, {
      sprite: spritemap,
      onend:onEndStereo, 
      onplay:onStartPlaying
    }).play
    const playRight = useSound(buttonSfx, {
      sprite: spritemap,
      stereo: -1.0, 
      onend:onEndAny, 
      onplay:onStartPlaying
    }).play

    const playLeft = useSound(buttonSfx, {
      stereo: 1.0,
      sprite: spritemap, 
      onend:onEndAny, 
      onplay:onStartPlaying
    }).play
    const playBoth = (idR,idL,questionName) => {
      state.currentQuestion=questionName
      if (!state.hasPlayedDichotic[state.currentQuestion]){//on vérifie que le son n'a pas déjà été joué
        playRight({ id: "English_S01_M01_1" }) //PL DEBUG idR
        playLeft({ id: "English_S01_M01_1" }) //PL DEBUG idL
        state.hasPlayedDichotic[state.currentQuestion]=false //PL DEBUG true
        state.isPlayingDichotic[state.currentQuestion]=false //PL DEBUG true
      }

    }

    const change = (questionName, currentTrial) => {
      state.currentQuestion=questionName
//      if (state.willPlay<questions[questionName].all.length){ //on détermine quel type de son émettre. Vu qu'on commence par la stéréo, on regarde si le bon nombre d'essais stereo a été accompli
//        console.log(questions[questionName].all[state.willPlay.questionName])
        playStereo({id: questions[questionName].all[(state.willPlay[questionName]||0)]})
//      }
//      else{
//        console.log('tout a déjà été joué')
//      }

    }
    

    return {
      playBoth, answers, step:ref(1), questions, change, playing, state
    }
  },
  name: 'IndexPage'
})

</script>
